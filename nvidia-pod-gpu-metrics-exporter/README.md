<!-- # NVIDIA Pod GPU Metrics Exporter 설치 가이드 -->

# 구성 요소 및 버전

* dcgm-exporter([docker.io/nvidia/dcgm-exporter:2.1.4-2.2.0-ubuntu18.04](https://hub.docker.com/layers/nvidia/dcgm-exporter/2.1.4-2.2.0-ubuntu18.04/images/sha256-03d0156f5aba94e9627a185d78f5aff960e2eb6fc012194b3f87349932072360?context=explore))

# Prerequisites
1. NVIDIA driver가 설치되어있어야 합니다.
    * [NVIDIA Device Plugin 가이드 참고](../nvidia-device-plugin/README.md#step-0-nvidia-driver-설치)
2. nvidia-docker가 설치되어있어야 합니다.
	* [NVIDIA Device Plugin 가이드 참고](../nvidia-device-plugin/README.md#step-1-nvidia-docker2-설치)
3. Prometheus가 설치되어있어야 합니다.
	* Prometheus 가이드 참고
4. 필요한 yaml을 미리 다운로드합니다.
	* 작업 디렉토리 생성 및 환경 설정
        ```bash
		$ export EXPORTER_HOME=~/nvida-exporter-install
        $ mkdir -p ${EXPORTER_HOME}
        $ cd ${EXPORTER_HOME}
        ```
	* 필요한 yaml을 다운로드 합니다.
		```bash
		$ export VERSION_NAME=main # main 혹은 4.1 등 설치를 위한 version(git branch 명)을 사용
		$ wget https://raw.githubusercontent.com/tmax-cloud/install-nvidia-gpu-infra/${VERSION_NAME}/nvidia-pod-gpu-metrics-exporter/manifest/pod-gpu-metrics-exporter-daemonset.yaml
		$ wget https://raw.githubusercontent.com/tmax-cloud/install-nvidia-gpu-infra/${VERSION_NAME}/nvidia-pod-gpu-metrics-exporter/manifest/pod-gpu-metrics-exporter-service.yaml
		$ wget https://raw.githubusercontent.com/tmax-cloud/install-nvidia-gpu-infra/${VERSION_NAME}/nvidia-pod-gpu-metrics-exporter/manifest/pod-gpu-metrics-exporter-servicemonitor.yaml
		```

# 폐쇄망 구축 가이드
폐쇄망에서 설치를 진행해야 하는 경우 필요한 추가 작업에 대해 기술합니다.
1. **폐쇄망에서 설치하는 경우** 사용하는 image repository에 NVIDIA Pod GPU Metrics Exporter 설치 시 필요한 이미지를 push한다.
	```bash
	$ cd ${EXPORTER_HOME}
	$ export DCGM_EXPORTER_VERSION=2.1.4-2.2.0-ubuntu18.04
	$ docker pull nvidia/dcgm-exporter:${DCGM_EXPORTER_VERSION}
	$ docker save nvidia/dcgm-exporter:${DCGM_EXPORTER_VERSION} > dcgm-exporter_${DCGM_EXPORTER_VERSION}.tar
	```

2. 위의 과정에서 생성한 tar 파일들을 폐쇄망 환경으로 이동시킨 뒤 사용하려는 registry에 이미지를 push한다.
    ```bash
	$ docker load < dcgm-exporter_${DCGM_EXPORTER_VERSION}.tar

    # export REGISTRY={registry name}
	$ docker tag nvidia/dcgm-exporter:${DCGM_EXPORTER_VERSION} ${REGISTRY}/dcgm-exporter:${DCGM_EXPORTER_VERSION}

	$ docker push ${REGISTRY}/dcgm-exporter:${DCGM_EXPORTER_VERSION}
    ```

# 설치 가이드

## Step 0. NVIDIA GPU node에 label 추가
* 목적 : `NVIDIA GPU를 사용하는 node들에 label을 추가합니다`
* 생성 순서 : 
    * NVIDIA GPU node에 label 추가
	    ```bash
		$ kubectl label nodes {GPU node name} tmax/gpudriver=nvidia
		```

## Step 1. GPU Metrics Exporter DaemonSet 배포
* 목적 : `GPU Metrics Exporter DaemonSet을 배포합니다.`
* 생성 순서 :
    * GPU Metrics Exporter DaemonSet 배포
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl create -f pod-gpu-metrics-exporter-daemonset.yaml
		```
* 비고 :
    * `폐쇄망에서 설치를 진행하여 별도의 image registry를 사용하는 경우 registry 정보를 추가로 설정해준다.`
        ```bash
        $ cd ${EXPORTER_HOME}
        $ sed -i 's/nvidia\/pod-gpu-metrics-exporter/'${REGISTRY}'\/pod-gpu-metrics-exporter/g' pod-gpu-metrics-exporter-daemonset.yaml
        ```

## Step 2. GPU Metrics Exporter Service 배포
* 목적 : `GPU Metrics Exporter Service를 배포합니다.`
* 생성 순서 :
    * GPU Metrics Exporter Service 배포
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl create -f pod-gpu-metrics-exporter-service.yaml
		```

## Step 3. GPU Metrics Exporter ServiceMonitor 배포
* 목적 : `GPU Metrics Exporter ServiceMonitor를 배포합니다.`
* 생성 순서 :
    * GPU Metrics Exporter ServiceMonitor 배포
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl create -f pod-gpu-metrics-exporter-servicemonitor.yaml
		```

## Step 4. Metric 확인
* 목적 : `NVIDIA GPU Metric이 제대로 출력되는 지 확인합니다.`
* 생성 순서 : 
	* Exporter로부터 Metric을 확인
		```bash
		$ curl -sL http://{Service IP}:9400/metrics
		# HELP DCGM_FI_DEV_SM_CLOCK SM clock frequency (in MHz).
		# TYPE DCGM_FI_DEV_SM_CLOCK gauge
		# HELP DCGM_FI_DEV_MEM_CLOCK Memory clock frequency (in MHz).
		# TYPE DCGM_FI_DEV_MEM_CLOCK gauge
		# HELP DCGM_FI_DEV_MEMORY_TEMP Memory temperature (in C).
		# TYPE DCGM_FI_DEV_MEMORY_TEMP gauge
		# HELP DCGM_FI_DEV_GPU_TEMP GPU temperature (in C).
		# TYPE DCGM_FI_DEV_GPU_TEMP gauge
		# HELP DCGM_FI_DEV_POWER_USAGE Power draw (in W).
		# TYPE DCGM_FI_DEV_POWER_USAGE gauge
		# HELP DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION Total energy consumption since boot (in mJ).
		# TYPE DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION counter
		# HELP DCGM_FI_DEV_PCIE_TX_THROUGHPUT Total number of bytes transmitted through PCIe TX (in KB) via NVML.
		# TYPE DCGM_FI_DEV_PCIE_TX_THROUGHPUT counter
		# HELP DCGM_FI_DEV_PCIE_RX_THROUGHPUT Total number of bytes received through PCIe RX (in KB) via NVML.
		# TYPE DCGM_FI_DEV_PCIE_RX_THROUGHPUT counter
		# HELP DCGM_FI_DEV_PCIE_REPLAY_COUNTER Total number of PCIe retries.
		# TYPE DCGM_FI_DEV_PCIE_REPLAY_COUNTER counter
		# HELP DCGM_FI_DEV_GPU_UTIL GPU utilization (in %).
		# TYPE DCGM_FI_DEV_GPU_UTIL gauge
		# HELP DCGM_FI_DEV_MEM_COPY_UTIL Memory utilization (in %).
		# TYPE DCGM_FI_DEV_MEM_COPY_UTIL gauge
		# HELP DCGM_FI_DEV_ENC_UTIL Encoder utilization (in %).
		# TYPE DCGM_FI_DEV_ENC_UTIL gauge
		# HELP DCGM_FI_DEV_DEC_UTIL Decoder utilization (in %).
		# TYPE DCGM_FI_DEV_DEC_UTIL gauge
		# HELP DCGM_FI_DEV_XID_ERRORS Value of the last XID error encountered.
		# TYPE DCGM_FI_DEV_XID_ERRORS gauge
		# HELP DCGM_FI_DEV_POWER_VIOLATION Throttling duration due to power constraints (in us).
		# TYPE DCGM_FI_DEV_POWER_VIOLATION counter
		# HELP DCGM_FI_DEV_THERMAL_VIOLATION Throttling duration due to thermal constraints (in us).
		# TYPE DCGM_FI_DEV_THERMAL_VIOLATION counter
		# HELP DCGM_FI_DEV_SYNC_BOOST_VIOLATION Throttling duration due to sync-boost constraints (in us).
		# TYPE DCGM_FI_DEV_SYNC_BOOST_VIOLATION counter
		# HELP DCGM_FI_DEV_BOARD_LIMIT_VIOLATION Throttling duration due to board limit constraints (in us).
		# TYPE DCGM_FI_DEV_BOARD_LIMIT_VIOLATION counter
		# HELP DCGM_FI_DEV_LOW_UTIL_VIOLATION Throttling duration due to low utilization (in us).
		# TYPE DCGM_FI_DEV_LOW_UTIL_VIOLATION counter
		# HELP DCGM_FI_DEV_RELIABILITY_VIOLATION Throttling duration due to reliability constraints (in us).
		# TYPE DCGM_FI_DEV_RELIABILITY_VIOLATION counter
		# HELP DCGM_FI_DEV_FB_FREE Framebuffer memory free (in MiB).
		# TYPE DCGM_FI_DEV_FB_FREE gauge
		# HELP DCGM_FI_DEV_FB_USED Framebuffer memory used (in MiB).
		# TYPE DCGM_FI_DEV_FB_USED gauge
		# HELP DCGM_FI_DEV_ECC_SBE_VOL_TOTAL Total number of single-bit volatile ECC errors.
		# TYPE DCGM_FI_DEV_ECC_SBE_VOL_TOTAL counter
		# HELP DCGM_FI_DEV_ECC_DBE_VOL_TOTAL Total number of double-bit volatile ECC errors.
		# TYPE DCGM_FI_DEV_ECC_DBE_VOL_TOTAL counter
		# HELP DCGM_FI_DEV_ECC_SBE_AGG_TOTAL Total number of single-bit persistent ECC errors.
		# TYPE DCGM_FI_DEV_ECC_SBE_AGG_TOTAL counter
		# HELP DCGM_FI_DEV_ECC_DBE_AGG_TOTAL Total number of double-bit persistent ECC errors.
		# TYPE DCGM_FI_DEV_ECC_DBE_AGG_TOTAL counter
		# HELP DCGM_FI_DEV_RETIRED_SBE Total number of retired pages due to single-bit errors.
		# TYPE DCGM_FI_DEV_RETIRED_SBE counter
		# HELP DCGM_FI_DEV_RETIRED_DBE Total number of retired pages due to double-bit errors.
		# TYPE DCGM_FI_DEV_RETIRED_DBE counter
		# HELP DCGM_FI_DEV_RETIRED_PENDING Total number of pages pending retirement.
		# TYPE DCGM_FI_DEV_RETIRED_PENDING counter
		# HELP DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL Total number of NVLink flow-control CRC errors.
		# TYPE DCGM_FI_DEV_NVLINK_CRC_FLIT_ERROR_COUNT_TOTAL counter
		# HELP DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL Total number of NVLink data CRC errors.
		# TYPE DCGM_FI_DEV_NVLINK_CRC_DATA_ERROR_COUNT_TOTAL counter
		# HELP DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL Total number of NVLink retries.
		# TYPE DCGM_FI_DEV_NVLINK_REPLAY_ERROR_COUNT_TOTAL counter
		# HELP DCGM_FI_DEV_NVLINK_RECOVERY_ERROR_COUNT_TOTAL Total number of NVLink recovery errors.
		# TYPE DCGM_FI_DEV_NVLINK_RECOVERY_ERROR_COUNT_TOTAL counter
		# HELP DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL Total number of NVLink bandwidth counters for all lanes
		# TYPE DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL counter
		# HELP DCGM_FI_DEV_VGPU_LICENSE_STATUS vGPU License status
		# TYPE DCGM_FI_DEV_VGPU_LICENSE_STATUS gauge
		# HELP DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS Number of remapped rows for uncorrectable errors
		# TYPE DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS counter
		# HELP DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS Number of remapped rows for correctable errors
		# TYPE DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS counter
		# HELP DCGM_FI_DEV_ROW_REMAP_FAILURE Whether remapping of rows has failed
		# TYPE DCGM_FI_DEV_ROW_REMAP_FAILURE gauge


		DCGM_FI_DEV_SM_CLOCK{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 139
		DCGM_FI_DEV_MEM_CLOCK{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 405
		DCGM_FI_DEV_GPU_TEMP{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 32
		DCGM_FI_DEV_POWER_USAGE{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 5.208000
		DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 7299606661
		DCGM_FI_DEV_PCIE_REPLAY_COUNTER{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_GPU_UTIL{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_MEM_COPY_UTIL{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_ENC_UTIL{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_DEC_UTIL{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_XID_ERRORS{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_POWER_VIOLATION{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_THERMAL_VIOLATION{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_SYNC_BOOST_VIOLATION{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_FB_FREE{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 5059
		DCGM_FI_DEV_FB_USED{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_NVLINK_BANDWIDTH_TOTAL{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		DCGM_FI_DEV_VGPU_LICENSE_STATUS{gpu="0",UUID="GPU-c0b5694b-2b5d-6b20-f903-558341437f6b",device="nvidia0"} 0
		```

# 삭제 가이드

## Step 1. GPU Metrics Exporter ServiceMonitor 삭제
* 목적 : `GPU Metrics Exporter ServiceMonitor를 삭제합니다.`
* 삭제 순서 :
    * GPU Metrics Exporter ServiceMonitor 삭제
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl delete -f pod-gpu-metrics-exporter-servicemonitor.yaml
		```

## Step 2. GPU Metrics Exporter Service 삭제
* 목적 : `GPU Metrics Exporter Service를 삭제합니다.`
* 삭제 순서 :
    * GPU Metrics Exporter Service 삭제
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl delete -f pod-gpu-metrics-exporter-service.yaml
		```

## Step 3. GPU Metrics Exporter DaemonSet 삭제
* 목적 : `GPU Metrics Exporter DaemonSet을 삭제합니다.`
* 삭제 순서 :
    * GPU Metrics Exporter DaemonSet 삭제
		```bash
		$ cd ${EXPORTER_HOME}
		$ kubectl delete -f pod-gpu-metrics-exporter-daemonset.yaml
		```

## Step 4. NVIDIA GPU node에 label 삭제
* 목적 : `NVIDIA GPU를 사용하는 node들에 label을 삭제합니다`
* 삭제 순서 : 
    * NVIDIA GPU node에 label 삭제
	    ```bash
		$ kubectl label nodes {GPU node name} tmax/gpudriver-
		```