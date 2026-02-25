# Nvidia-Actions-Runner (and Dind) for ARC controller on Kubernetes

CUDA 12 and Ubuntu 24.04 based Actions Runner image and Nvidia-DinD (Docker in Docker) image, optimized for use with https://github.com/actions/actions-runner-controller

## Usage With helm

```bash
helm upgrade -i "${RELEASE_NAME}" -f template.yaml \
    --namespace "${NAMESPACE}" \
    --create-namespace \
    --set githubConfigUrl="${GITHUB_CONFIG_URL}" \
    --set githubConfigSecret.github_token="${GITHUB_PAT}" \
    oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

### Nvidia Actions Runner Template
```yaml
template:
  spec:
    tolerations:
      - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    containers:
      - name: runner
        image: ltpn/nvidia-actions-runner:latest
        command:
          - /home/runner/run.sh
```

### Nvidia-DinD-Runner Template  

```yaml
template:
spec:
    tolerations:
    - key: "nvidia.com/gpu"
        operator: "Exists"
        effect: "NoSchedule"
    initContainers:
    - name: init-dind-externals
        image: ltpn/nvidia-actions-runner:latest
        command:
        ["cp", "-r", "-v", "/home/runner/externals/.", "/home/runner/tmpDir/"]
        volumeMounts:
        - name: dind-externals
            mountPath: /home/runner/tmpDir
    containers:
    - name: runner
        image: ltpn/nvidia-actions-runner:latest
        command: ["/home/runner/run.sh"]
        env:
        - name: DOCKER_HOST
            value: unix:///var/run/docker.sock
        volumeMounts:
        - name: work
            mountPath: /home/runner/_work
        - name: dind-sock
            mountPath: /var/run
            readOnly: true
    - name: dind
        image: ltpn/nvidia-dind:latest
        args:
        - dockerd
        - --host=unix:///var/run/docker.sock
        - --group=$(DOCKER_GROUP_GID)
        env:
        - name: DOCKER_GROUP_GID
            value: "123"
        securityContext:
        privileged: true
        volumeMounts:
        - name: work
            mountPath: /home/runner/_work
        - name: dind-sock
            mountPath: /var/run
        - name: dind-externals
            mountPath: /home/runner/externals
    volumes:
    - name: work
        emptyDir: {}
    - name: dind-sock
        emptyDir: {}
    - name: dind-externals
        emptyDir: {}
```

### Nvidia-DinD-Runner Template (Talos)

On Talos, with the [NVIDIA Device Plugin](https://docs.siderolabs.com/talos/latest/configure-your-talos-cluster/hardware-and-drivers/nvidia-gpu-proprietary#deploying-nvidia-device-plugin) installed, we can even just use a custom `dind` container image:
```yaml
template:
    spec:
      runtimeClassName: nvidia
      initContainers:
      - name: init-dind-externals
        image: ghcr.io/actions/actions-runner:latest
        command: ["cp", "-r", "/home/runner/externals/.", "/home/runner/tmpDir/"]
        volumeMounts:
          - name: dind-externals
            mountPath: /home/runner/tmpDir
      containers:
      - name: runner
        image: ghcr.io/actions/actions-runner:latest
        command: ["/home/runner/run.sh"]
        env:
          - name: DOCKER_HOST
            value: unix:///var/run/docker.sock
          - name: RUNNER_WAIT_FOR_DOCKER_IN_SECONDS
            value: "120"
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
      - name: dind
        image: ghcr.io/ltpn/nvidia-dind:latest
        args:
          - dockerd
          - --host=unix:///var/run/docker.sock
          - --group=$(DOCKER_GROUP_GID)
          - --default-runtime=nvidia
        env:
          - name: DOCKER_GROUP_GID
            value: "123"
        securityContext:
          privileged: true
        restartPolicy: Always
        startupProbe:
          exec:
            command:
              - docker
              - info
          initialDelaySeconds: 0
          failureThreshold: 24
          periodSeconds: 5
        volumeMounts:
          - name: work
            mountPath: /home/runner/_work
          - name: dind-sock
            mountPath: /var/run
          - name: dind-externals
            mountPath: /home/runner/externals
      volumes:
      - name: work
        emptyDir: {}
      - name: dind-sock
        emptyDir: {}
      - name: dind-externals
        emptyDir: {}
```

## Acknowledgements

- https://github.com/Extrality/nvidia-dind
- https://github.com/actions/runner
- https://github.com/actions/actions-runner-controller
- https://github.com/actions/actions-runner-controller/discussions/3409
- https://github.com/CuriousDolphin/nvidia-dind-actions-runners
