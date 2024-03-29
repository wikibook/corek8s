# A comprehensive Network Policy construction and testing library

This repo implements https://github.com/vmware-tanzu/antrea/blob/community-network-policy-tests/docs/design/cni-testing-initiative-upstream.md, a fast, comprehensive truth table matrix for network policy's which can be used to ensure that you're CNI provider is fast, reliably, and air-tight.

(Note this is a new repo, so some features not implemented yet, like Egress Builders)

## A super-simple builder for experimenting with and validating your own network policys

One hard thing about network policies is *testing* that they do *exactly* what you thought they did.  You can fork this repo and code up a network policy quickly, and in a few lines of code, verify that it works perfectly.

You can add a new test in just a few lines of code, for example, this test creates a network policy which ensures that 
only traffic from `b` pods in the 3 namespaces `x,y,z` can access a the `a` pod, which lives in namespace `x`.

```
	builder := &utils.NetworkPolicySpecBuilder{}
	builder = builder.SetName("allow-client-a-via-pod-selector").SetPodSelector(map[string]string{"pod": "a"})
	builder.SetTypeIngress()
	builder.AddIngress(nil, &p80, nil, nil, map[string]string{"pod": "b"}, map[string]string{"ns": "x"}, nil, nil)
	builder.AddIngress(nil, &p80, nil, nil, map[string]string{"pod": "b"}, map[string]string{"ns": "y"}, nil, nil)
	builder.AddIngress(nil, &p80, nil, nil, map[string]string{"pod": "b"}, map[string]string{"ns": "z"}, nil, nil)
	k8s.CreateNetworkPolicy("x", builder.Get())
	m.ExpectAllIngress("x","a",false)
	m.Expect("x", "b", "x", "a", true)
	m.Expect("y", "b", "x", "a", true)
	m.Expect("z", "b", "x", "a", true)
	m.Expect("x", "a", "x", "a", true)
```
This policy is then validated using the following three-liner:

```
	matrix := TestPodLabelWhitelistingFromBToA(&k8s)
	validate(&k8s, matrix)
	summary, pass := matrix.Summary()
	fmt.Println(summary, pass)
```

The output of these tests shows all probes, in the logs, so you can reproduce them, and also output the entire truth table of pod<->pod connectivity for you once the test is done. 

```
time="2020-02-18T20:09:31Z" level=info msg=".... Done"
time="2020-02-18T20:09:31Z" level=info msg="Running: kubectl exec -t -i zc-7655cf9dd6-jcpqv -n z -- wget -s -T 1 http://100.96.0.48:80"
time="2020-02-18T20:09:31Z" level=info msg=".... Done"
y_a
--> map[x_a:false x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
y_c
--> map[x_a:false x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
z_a
--> map[x_a:false x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
z_b
--> map[x_a:true x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
z_c
--> map[x_a:false x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
x_a
--> map[x_a:true x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
x_b
--> map[x_a:true x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
x_c
--> map[x_a:false x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
y_b
--> map[x_a:true x_b:true x_c:true y_a:true y_b:true y_c:true z_a:true z_b:true z_c:true]
correct:81, incorrect:0, result=%!(EXTRA bool=true) true

-	x/a	y/a	z/a	x/b	y/b	z/b	x/c	y/c	z/c
x/a	.	.	.	.	.	.	.	.	.
y/a	X	.	.	.	.	.	.	.	.
z/a	X	.	.	.	.	.	.	.	.
x/b	.	.	.	.	.	.	.	.	.
y/b	.	.	.	.	.	.	.	.	.
z/b	.	.	.	.	.	.	.	.	.
x/c	X	.	.	.	.	.	.	.	.
y/c	X	.	.	.	.	.	.	.	.
z/c	X	.	.	.	.	.	.	.	.


observed:

-	x/a	y/a	z/a	x/b	y/b	z/b	x/c	y/c	z/c
x/a	.	.	.	.	.	.	.	.	.
y/a	X	.	.	.	.	.	.	.	.
z/a	X	.	.	.	.	.	.	.	.
x/b	.	.	.	.	.	.	.	.	.
y/b	.	.	.	.	.	.	.	.	.
z/b	.	.	.	.	.	.	.	.	.
x/c	X	.	.	.	.	.	.	.	.
y/c	X	.	.	.	.	.	.	.	.
z/c	X	.	.	.	.	.	.	.	.


comparison:

-	x/a	y/a	z/a	x/b	y/b	z/b	x/c	y/c	z/c
x/a	.	.	.	.	.	.	.	.	.
y/a	.	.	.	.	.	.	.	.	.
z/a	.	.	.	.	.	.	.	.	.
x/b	.	.	.	.	.	.	.	.	.
y/b	.	.	.	.	.	.	.	.	.
z/b	.	.	.	.	.	.	.	.	.
x/c	.	.	.	.	.	.	.	.	.
y/c	.	.	.	.	.	.	.	.	.
z/c	.	.	.	.	.	.	.	.	.

```

## How is this different then NetworkPolicy tests in upstream K8s ?

We are working to merge this into upstream Kubernetes, in the meanwhile, here's the differences.

- We define tests as *truth tables, and have a 'builder' library* for building up network policy structs with almost no boilerplate, meaning you can define a very sophisticated network policy test in just a few lines of code.
- *Comprehensive:* All pod-to-pod connectivity is validated for every test run.  In a typical network policy test in current upstream we only validate 2 or 3 scenarios, leaving out intra and inner namespace connections which might be comprimised due to a hard to detect CNI inconsistency.  In these tests, we test all 81 connections for 3 identical pods running in 3 different namespaces (i.e. the 9x9 connectivity matrix).
- *Transparent:* Each test prints out a `kubectl` command you can run to re-probe a given pods connectivity patterns.
- It's *fast:* Because we use `kubectl exec` to run tests with `wget` between pods, all 81 tests can easily finish in 20 seconds or less, even if pod scheduling is slow.  This is because no polling is done, and there is no down/uptime for pods.
- *Easy to reason about:* The pods in this repo stay up forever, so you can reuse the above kubectl commands outputted by your netpol logs to exec into a pod and reproduce any failures.
- *Scalable:* If you want to test 32 policies, all at once ? Just take a look at the example test (in `main`) and copy paste a few lines, and you'll be testing enterprise CNI application patterns in a heartbeat.

## Users

### Test Your Dang CNI !  Now !

Create the policy probe tests... 

```
kubectl create clusterrolebinding netpol --clusterrole=admin --serviceaccount=kube-system:netpol
kubectl create sa netpol -n kube-system
kubectl create -f https://raw.githubusercontent.com/jayunit100/k8sprototypes/master/netpol/install.yml
```

Now, look at the results of the network policy probe... 

```
 kubectl logs `kubectl get pods -n kube-system | grep netpol | cut -d' ' -f 1` -n kube-system  
```
 
## Developers

Would love help with this!  If you want to get started hacking .... 
 
### Create a cluster if you dont have one  and run from source....
```
kind create cluster
kind get kubeconfig > ~/.kube/config
go build ./main
./main
```

### Alternate setup

```
kind create cluster --name netpols
kubectl cluster-info --context kind-netpols

git clone git@github.com:jayunit100/k8sprototypes.git 
cd k8sprototypes

cd kind
./kind-local-up.sh
cd ..

cd netpol
go run pkg/main/main.go
```
