# kubelet
- remoteRuntimeEndpoint = "unix:///var/run/dockershim.sock" : runtime的unix sock
- DockershimRootDirectory:   "/var/lib/dockershim",
- CNIBinDir:   "/opt/cni/bin",
- CNIConfDir:  "/etc/cni/net.d",
- CNICacheDir: "/var/lib/cni/cache",

- kube-controller-manager/app/plugins.go 

## depend
- VolumePlugins
- DynamicPluginProber
- DockerOptions


## PodSandbox

