# k8s-tls-proxy

An Nginx instance that acts as an HTTP proxy for TLS endpoints.

Some pods in the Kubernetes control plane (`hyperkube` and `podmaster`)
are not easy to use with TLS-enabled `etcd` endpoints. This Docker container
acts as a MITM proxy, offering a plain HTTP `etcd` endpoint.


## Usage

If you have a TLS-enabled `etcd` endpoint at `https://1.2.3.4:2379`,
for instance, then you could use the following YAML descriptor for the
Kubernetes API server:

    apiVersion: v1
	kind: Pod
    metadata:
      name: kube-apiserver
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
        - name: kube-apiserver
          image: gcr.io/google_containers/hyperkube:%(kubernetes_version)s
          command:
            - /hyperkube
            - apiserver
              - --etcd-servers="http://127.0.0.1:2379""
			  [..]
        - name: k8s-tls-proxy
		  image: carletes/k8s-tls-proxy
		  command:
		    - /tls-proxy
			- --remote-host=1.2.3.4
			- --remote-port=2379
			- --local-host=127.0.0.1
			- --local-port=2379
			- --cert-file=/etc/kubernetes/ssl/node.pem
			- --key-file=/etc/kubernetes/ssl/node-key.pem
			- --ca-file=/etc/kubernetes/ssl/ca.pem
	  volumes:
          - hostPath:
              path: /etc/kubernetes/ssl
            name: ssl-certs-kubernetes
          - hostPath:
              path: /usr/share/ca-certificates
            name: ssl-certs-host

Similarly, for the `podmaster` pods:

	apiVersion: v1
    kind: Pod
    metadata:
      name: kube-podmaster
      namespace: kube-system
    spec:
      hostNetwork: true
      containers:
        - name: scheduler-elector
          image: gcr.io/google_containers/podmaster:1.1
          command:
            - /podmaster
              - --key=scheduler
			  - --etcd-servers="http://127.0.0.1:2379"
			  [..]
	    - name: controller-manager-elector
          image: gcr.io/google_containers/podmaster:1.1
          command:
            - /podmaster
              - --key=controller
              - --etcd-servers="http://127.0.0.1:2379"
			  [..]
        - name: k8s-tls-proxy
		  image: carletes/k8s-tls-proxy
		  command:
		    - /tls-proxy
			- --remote-host=1.2.3.4
			- --remote-port=2379
			- --local-host=127.0.0.1
			- --local-port=2379
			- --cert-file=/etc/kubernetes/ssl/node.pem
			- --key-file=/etc/kubernetes/ssl/node-key.pem
			- --ca-file=/etc/kubernetes/ssl/ca.pem
