# always be git 😁
git init

# create empty cluster HelmRelease;
flux create helmrelease --export base-cluster -n flux-system --source HelmRepository/teuto-net.flux-system --chart base-cluster --chart-version 5.x.x > cluster.yaml

# maybe use the following name for your cluster;
kubectl get node -o jsonpath='{.items[0].metadata.annotations.cluster\.x-k8s\.io/cluster-name}'

# configure according to your needs, at least `.global.clusterName` is needed
# additionally, you should add your git repo to `.flux.gitRepositories`, see [the documentation](https://github.com/teutonet/teutonet-helm-charts/tree/main/charts/base-cluster#81--property-base-cluster-configuration--flux--gitrepositories)
# make sure to use the correct url format, see [the documentation](https://github.com/teutonet/teutonet-helm-charts/tree/main/charts/base-cluster#81112-property-base-cluster-configuration--flux--gitrepositories--additionalproperties--allof--item-0--oneof--item-1)
vi cluster.yaml

# create HelmRelease for flux to manage itself
kubectl create namespace flux-system --dry-run=client -o yaml > flux.yaml
flux create source helm --url https://fluxcd-community.github.io/helm-charts flux -n flux-system --export >> flux.yaml
flux create helmrelease --export flux -n flux-system --source HelmRepository/flux.flux-system --chart flux2 --chart-version 2.x.x >> flux.yaml

# add, commit and push resources
git add cluster.yaml flux.yaml
git commit cluster.yaml flux.yaml
git push

# after this you should be on the KUBECONFIG for the cluster
# we explicitly do not use `flux bootstrap` or `flux install` as this creates kustomization stuff and installs flux manually
kubectl apply --server-side -f flux.yaml # ignore the errors about missing CRDs
helm install -n flux-system flux flux2 --repo https://fluxcd-community.github.io/helm-charts --version 2.x.x --atomic

# manual initial installation of the chart, afterwards the chart takes over
# after the installation finished, follow the on-screen instructions to configure your flux, distribute KUBECONFIGs, ...
helm install -n flux-system base-cluster oci://ghcr.io/teutonet/teutonet-helm-charts/base-cluster --version 4.x.x --atomic --values <(cat cluster.yaml | yq -y .spec.values)

# you can use this command to get the instructions again
# e.g. when adding users, gitRepositories, ...
helm -n flux-system get notes base-cluster