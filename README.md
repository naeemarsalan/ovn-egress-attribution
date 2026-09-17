# Tracking down egress traffic to an IP range on OpenShift

Find out which namespaces and pods are still reaching an address range, before
you switch it off.

Pod IPs get recycled in seconds, so looking up "who owns this IP" after the fact
can name a pod that exited long ago. This avoids that by taking both halves from
OVN: **what talked** (logged at enforcement time) and **who held the address
then** (OVN's own IP bind/release records), joined on IP *and time*.

No node access, no privileged pods, no new operator.

---

## Step 1 — Check your cluster is ready

```bash
oc get network.operator cluster -o jsonpath='{.spec.defaultNetwork.type}{"\n"}'
oc api-resources | grep adminnetworkpolic
oc get anp -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority
```

You want `OVNKubernetes` and a line for `adminnetworkpolicies` — note which API
version it prints and use that below.

If any AdminNetworkPolicy already exists, read it. One at a higher priority that
terminally allows or denies your ranges means yours never fires, and you get an
empty report with no error at all.

## Step 2 — Raise the audit rate limit

Do this **first**, before the window you want to measure. It restarts every
`ovnkube-node` pod, and the lease records you need only live in container output
— a restart destroys them.

The default is 20 log lines/sec/node and it drops the rest silently. A completed
TCP connection produces 6 lines, so the default is spent by about 3
connections/sec. Measured on 40 namespaces sending 1000 connections: the default
captured 167, and **2 namespaces that were actively sending never appeared in
the report at all**.

```bash
oc patch network.operator cluster --type=merge \
  -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"policyAuditConfig":{"rateLimit":500}}}}}'

# wait for the rollout - watch the clusteroperator, not the pods
sleep 30
until [ "$(oc get co network -o jsonpath='{.status.conditions[?(@.type=="Progressing")].status}')" = "False" ] \
   && [ "$(oc get co network -o jsonpath='{.status.conditions[?(@.type=="Degraded")].status}')" = "False" ]; do
  sleep 15
done
oc get co network
```

### Working out the number

500 is a starting point, not the answer. The limit is **per node, per second**,
and what consumes it is log lines, not connections:

```
rateLimit  >=  L x C_node x H

  L       = 6      log lines per completed TCP connection (measured, exact)
  C_node  =        peak connections/sec to your ranges on the BUSIEST node
  H       = 3..4   burst headroom - the meter has burst_size 0, so the
                   instantaneous rate matters, not the average
```

Estimating `C_node` from namespaces:

```
C_node  =  (N x r) x skew

  N      = namespaces that reach the target ranges
  r      = peak connections/sec each one makes
  skew   = share landing on the busiest node.
           1/nodes if evenly spread; use 2/nodes to be safe, because
           pods are not evenly spread and short-lived ones cluster.
```

Worked from the measured run — 40 namespaces, 25 connections each, over
2 worker nodes, completing in ~24s:

```
N x r    = 1000 conns / 24s        = 42 conns/sec cluster-wide
skew     = 2/2 nodes               = 1.0   (worst case, all on one node)
C_node   = 42 x 1.0                = 42 conns/sec
rateLimit>= 6 x 42 x 4             = 1008
```

At 500 that load captured 1000/1000, so the headroom factor is generous —
which is the point. Round up; the cost of being wrong is silent data loss.

### Validating the number you chose

Measure what each node is actually emitting and compare:

```bash
for p in $(oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-node -o name | cut -d/ -f2); do
  a=$(oc exec -n openshift-ovn-kubernetes $p -c ovn-controller -- sh -c 'wc -l < /var/log/ovn/acl-audit-log.log' 2>/dev/null)
  sleep 60
  b=$(oc exec -n openshift-ovn-kubernetes $p -c ovn-controller -- sh -c 'wc -l < /var/log/ovn/acl-audit-log.log' 2>/dev/null)
  echo "$p: $(( (b-a) / 60 )) lines/sec  (limit $(oc get network.operator cluster -o jsonpath='{.spec.defaultNetwork.ovnKubernetesConfig.policyAuditConfig.rateLimit}'))"
done
```

**Careful how you read this.** If a node's observed rate is sitting close to the
configured limit, you are measuring the *limit*, not the demand — the real
demand could be any amount higher and you cannot tell from this number alone.

The way out is a saturation test: raise the limit and measure again.

```
observed rate goes up   -> you were capped. Raise again and repeat.
observed rate unchanged -> you have found the real demand. Add headroom and stop.
```

Two consecutive increases with no change in the observed rate means the limit
is no longer the constraint. That, not a calculation, is what tells you the
number is big enough.

## Step 3 — Apply the policy

`action: Pass` hands off to NetworkPolicy, so no packet's fate changes — this
only turns on logging. Don't use `Allow`; it's terminal and would override an
existing NetworkPolicy.

Start with a **reachable** test address so you can verify it works. Your real
ranges go in at Step 8.

```yaml
# anp.yaml
apiVersion: policy.networking.k8s.io/v1alpha1
kind: AdminNetworkPolicy
metadata:
  name: egress-audit
  annotations:
    k8s.ovn.org/acl-logging: '{"pass":"notice"}'
spec:
  priority: 50
  subject:
    namespaces: {}                # whole cluster, including new namespaces
  egress:
  - name: target
    action: Pass
    to:
    - networks:
      - 1.1.1.1/32
```

```bash
oc apply -f anp.yaml
oc get anp egress-audit -o jsonpath='{.status.conditions[*].status}{"\n"}'
```

Every zone should say `True`. If one doesn't, you'll get partial data that looks
complete.

## Step 4 — Verify it's actually recording

Send traffic and go find it.

```bash
oc create ns egress-audit-test

oc -n egress-audit-test run probe --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never --command -- \
  sh -c 'for i in 1 2 3 4 5; do curl -s -o /dev/null --max-time 5 http://1.1.1.1/; sleep 1; done'

until [ "$(oc -n egress-audit-test get pod probe -o jsonpath='{.status.phase}')" = "Succeeded" ]; do sleep 3; done

NODE=$(oc -n egress-audit-test get pod probe -o jsonpath='{.spec.nodeName}')
OVNPOD=$(oc get pod -n openshift-ovn-kubernetes -l app=ovnkube-node \
  --field-selector spec.nodeName=$NODE -o jsonpath='{.items[0].metadata.name}')

oc exec -n openshift-ovn-kubernetes $OVNPOD -c ovn-controller -- \
  sh -c 'cat /var/log/ovn/acl-audit-log.log*' | grep 'ANP:egress-audit:' | tail -3
```

You should see:

```
...|acl_log(ovn_pinctrl0)|INFO|name="ANP:egress-audit:Egress:0", verdict=pass,
direction=from-lport: tcp,...,nw_src=10.128.2.87,nw_dst=1.1.1.1,...,tcp_flags=syn
```

Nothing there? Go back to Step 1 — a conflicting higher-priority policy is
almost always the reason.

## Step 5 — Check you're not silently losing records

Everything downstream depends on this. Send a known number of connections and
count what comes back.

**Pace them.** A tight back-to-back loop measures how the meter handles a burst,
not steady-state capture, and reports ~75% no matter what the rate limit is.

```bash
BASE=$(oc exec -n openshift-ovn-kubernetes $OVNPOD -c ovn-controller -- \
       sh -c 'cat /var/log/ovn/acl-audit-log.log*' 2>/dev/null \
       | grep 'ANP:egress-audit:' | grep -c 'tcp_flags=syn')

oc -n egress-audit-test run counter --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never --overrides="{\"spec\":{\"nodeName\":\"$NODE\"}}" --command -- \
  sh -c 'ok=0; for i in $(seq 1 100); do curl -s -o /dev/null --max-time 5 http://1.1.1.1/ && ok=$((ok+1)); sleep 0.3; done; echo SENT=$ok'

until [ "$(oc -n egress-audit-test get pod counter -o jsonpath='{.status.phase}')" = "Succeeded" ]; do sleep 5; done
sleep 10

SENT=$(oc -n egress-audit-test logs counter | grep -oP 'SENT=\K[0-9]+')
NOW=$(oc exec -n openshift-ovn-kubernetes $OVNPOD -c ovn-controller -- \
      sh -c 'cat /var/log/ovn/acl-audit-log.log*' 2>/dev/null \
      | grep 'ANP:egress-audit:' | grep -c 'tcp_flags=syn')
echo "sent=$SENT captured=$((NOW - BASE))"
```

You want essentially all 100. Below ~95% means records are being dropped — raise
`rateLimit` again (Step 2) and repeat. Remember `$OVNPOD` changes name after
each restart, so look it up again.

Worth also confirming the attribution is *right*, not just present:

```bash
SRC=$(oc exec -n openshift-ovn-kubernetes $OVNPOD -c ovn-controller -- \
      sh -c 'cat /var/log/ovn/acl-audit-log.log*' 2>/dev/null \
      | grep 'ANP:egress-audit:' | grep 'tcp_flags=syn' | tail -1 | grep -oP 'nw_src=\K[\d.]+')
oc logs -n openshift-ovn-kubernetes $OVNPOD -c ovnkube-controller --since=30m | grep "IPs: \[$SRC/"
```

```
ConfigureOVS: namespace: egress-audit-test, podName: counter, ... IPs: [10.128.2.87/23]
```

That should name the pod you just ran.

## Step 6 — Collect the logs

Two streams, both from the `ovnkube-node` pods.

```bash
mkdir -p collected && : > collected/acl.log && : > collected/leases.log

for p in $(oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-node -o name | cut -d/ -f2); do
  # what talked - read the FILE, not oc logs
  oc exec -n openshift-ovn-kubernetes $p -c ovn-controller -- \
    sh -c 'cat /var/log/ovn/acl-audit-log.log* 2>/dev/null' 2>/dev/null \
    | grep 'acl_log' >> collected/acl.log

  # who held which IP, when - reach back further than the ACL window
  oc logs -n openshift-ovn-kubernetes $p -c ovnkube-controller --since=96h \
    | grep -E 'ConfigureOVS: namespace:|Attempting to release IPs for pod:' \
    >> collected/leases.log
done

wc -l collected/acl.log collected/leases.log
```

> Read the ACL log from the file, not `oc logs`. The file is a hostPath mount
> and survives pod restarts; container output doesn't. Right after a `rateLimit`
> restart, `oc logs` returned 0 records where the file held 2183.

The two streams look like this:

```
# collected/acl.log  -  what talked
...|acl_log(ovn_pinctrl0)|INFO|name="ANP:egress-audit:Egress:0", verdict=pass,
direction=from-lport: tcp,...,nw_src=10.128.2.87,nw_dst=1.1.1.1,...,tcp_flags=syn

# collected/leases.log  -  who held the address
ConfigureOVS: namespace: app-one, podName: worker-1, UID: "d66d…", IPs: [10.128.2.87/23]
Attempting to release IPs for pod: app-one/worker-1, ips: 10.128.2.87
```

If you have ACL records but **no** lease records, the `ovnkube-node` pods
restarted since the traffic happened and took the lease history with them.
Nothing can be attributed — run the window again without touching `rateLimit`.

## Step 7 — Parse them into a report

```python
#!/usr/bin/env python3
"""report.py - join ACL records to IP leases on IP AND time."""
import collections, datetime, re, sys

ACL  = re.compile(r'^(?P<ts>\S+?)\|.*?nw_src=(?P<src>[\d.]+),nw_dst=(?P<dst>[\d.]+)'
                  r'.*?tp_dst=(?P<dport>\d+),tcp_flags=(?P<f>[a-z|]+)')
BIND = re.compile(r'ConfigureOVS: namespace: (?P<ns>[^,]+), podName: (?P<pod>[^,]+),'
                  r'.*?IPs: \[(?P<ips>[^\]]+)\]')
REL  = re.compile(r'Attempting to release IPs for pod: (?P<ns>[^/]+)/(?P<pod>\S+?), ips: (?P<ips>\S+)')
KTS  = re.compile(r'^[IWEF](?P<md>\d{4}) (?P<t>\d{2}:\d{2}:\d{2}\.\d+)')
YEAR = datetime.date.today().year          # these log lines carry no year

def klog(line):
    m = KTS.match(line)
    if not m: return None
    mo, d = int(m.group('md')[:2]), int(m.group('md')[2:])
    hh, mm, rest = m.group('t').split(':'); ss, us = rest.split('.')
    return datetime.datetime(YEAR, mo, d, int(hh), int(mm), int(ss), int(us[:6]),
                             tzinfo=datetime.timezone.utc)

# build: for each IP, who held it and between when and when
ev = collections.defaultdict(list)
for line in open(sys.argv[2], errors='replace'):
    t = klog(line)
    if not t: continue
    m = BIND.search(line) or REL.search(line)
    if not m: continue
    kind = 'bind' if 'ConfigureOVS' in line else 'release'
    for ip in m.group('ips').split(','):
        ev[ip.strip().split('/')[0]].append(
            (t, kind, m.group('ns').strip(), m.group('pod').strip()))

leases = collections.defaultdict(list)
for ip, evs in ev.items():
    evs.sort(key=lambda x: x[0]); open_ = {}
    for t, kind, ns, pod in evs:
        if kind == 'bind': open_.setdefault((ns, pod), t)
        elif (ns, pod) in open_:
            leases[ip].append({'s': open_.pop((ns, pod)), 'e': t, 'ns': ns, 'pod': pod})
    for (ns, pod), s in open_.items():
        leases[ip].append({'s': s, 'e': None, 'ns': ns, 'pod': pod})
    leases[ip].sort(key=lambda x: x['s'])

floor = min((l['s'] for v in leases.values() for l in v), default=None)
rows, stale, bad = [], 0, 0
for line in open(sys.argv[1], errors='replace'):
    m = ACL.search(line.strip())
    if not m or m.group('f') != 'syn': continue     # one record per connection
    t = datetime.datetime.fromisoformat(m.group('ts').replace('Z', '+00:00'))
    hit = next((l for l in leases.get(m.group('src'), [])
                if l['s'] <= t and (l['e'] is None or t <= l['e'])), None)  # <- the time bound
    if hit: rows.append((hit['ns'], hit['pod']))
    elif floor and t < floor: stale += 1
    else: bad += 1

print(f"connections observed                       : {len(rows) + stale + bad}")
print(f"attributed to a namespace                  : {len(rows)}")
print(f"unattributed (predates lease window)       : {stale}")
print(f"unattributed (INSIDE window - investigate) : {bad}\n")
ns = collections.Counter(r[0] for r in rows)
pods = collections.defaultdict(set)
for r in rows: pods[r[0]].add(r[1])
print(f"{'NAMESPACE':<34}{'CONNECTIONS':>12}   PODS")
print("-" * 76)
for n, c in ns.most_common():
    p = ', '.join(sorted(pods[n])[:3]) + (f" (+{len(pods[n])-3})" if len(pods[n]) > 3 else "")
    print(f"{n:<34}{c:>12}   {p}")
```

```bash
python3 report.py collected/acl.log collected/leases.log
```

```
connections observed                       : 113
attributed to a namespace                  : 113
unattributed (predates lease window)       : 0
unattributed (INSIDE window - investigate) : 0

NAMESPACE                          CONNECTIONS   PODS
----------------------------------------------------------------------------
legacy-app                                 113   app-1, app-2, app-3 (+20)
```

`predates lease window` is fine — ACL records older than the oldest lease you
collected. `INSIDE window` should be zero; anything else means the lease log
didn't cover the traffic.

## Step 8 — Point it at your real ranges

Scope never changes — only the address list.

```bash
oc delete ns egress-audit-test

oc patch anp egress-audit --type=json -p '[{"op":"replace","path":"/spec/egress/0/to","value":[{"networks":["203.0.113.0/24","198.51.100.0/24"]}]}]'

oc get anp egress-audit -o jsonpath='{.spec.egress[0].to}{"\n"}'
```

Get whoever owns the endpoint to confirm the list is complete — an address
missing here is invisible to everything above, and you'd never know.

Real traffic is heavier than your test pod was, so re-run the validation from
[Step 2](#validating-the-number-you-chose) now that the real ranges are in
place. Raise the limit and re-measure until the observed rate stops climbing.

Then let it run as long as you need — days, if what you're hunting is
infrequent — and repeat Steps 6 and 7.

---

## Worth knowing

- **A conflicting policy gives you silence, not an error.** If another ANP at a
  higher priority terminally allows or denies your ranges, yours never fires and
  the report comes back empty. Check with
  `oc get anp -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority`.
- **Check your capture rate before trusting a clean result.** Send a known
  number of paced connections, count the `tcp_flags=syn` records, compare. Pace
  them — a tight loop measures the meter's burst handling (`burst_size` is 0)
  and reports ~75% no matter what the rate limit is.
- **Cross-check against the endpoint's own logs.** If the server saw 10,000
  requests and you captured 2,000, your clean namespaces are unobserved, not
  clean.
- **The lease lines are internal log messages, not an API.** An update can
  change their wording, and the report would then return nothing rather than
  failing. Alert if `grep -c 'ConfigureOVS: namespace:'` hits zero.
- **Pod network only.** hostNetwork pods and node-level processes don't cross a
  pod switch port. Check `Proxy/cluster` and MachineConfig separately.

## When you want to block instead of watch

Same object, one field:

```bash
oc patch anp egress-audit --type=json \
  -p '[{"op":"replace","path":"/spec/egress/0/action","value":"Deny"}]'
```

Keep the logging annotation on so anything you missed shows up failing rather
than disappearing. `Pass` puts it back.

## Removing it

```bash
oc delete anp egress-audit
oc patch network.operator cluster --type=merge \
  -p '{"spec":{"defaultNetwork":{"ovnKubernetesConfig":{"policyAuditConfig":{"rateLimit":20}}}}}'
```
