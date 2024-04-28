load('ext://helm_resource', 'helm_resource', 'helm_repo')

# Install app
k8s_yaml(helm('charts/metallb', name="metallb"))
k8s_yaml(helm('charts/dashboard', name="dashboard"))
k8s_yaml(helm('charts/bookinfo', name="bookinfo"))
k8s_yaml(helm('charts/metrics', name="metrics"))
k8s_yaml(helm('charts/istio-policies', name="istio-policies"))
k8s_yaml(helm('charts/debug', name="debug"))
