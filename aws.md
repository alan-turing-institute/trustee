#  TEE Evaluation on AWS

## TODO
- SEV-SNP instances
- EKS Nitro enclaves
  - Test how to deploy these
- EKS confidential containers
  - Confidential nodepool
  - pods vs nodes as TEEs

## References

- [Amazon EC2 instance attestation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nitrotpm-attestation.html)
- [Build the sample Amazon Linux 2023 image description](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/build-sample-ami.html)
- [Requirements for using NitroTPM with Amazon EC2 instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enable-nitrotpm-prerequisites.html#nitrotpm-instancetypes)
- [https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/prepare-attestation-service.html](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/prepare-attestation-service.html)
- [Demystify attestation: Cryptographically verify execution environment (CMP317)](https://www.youtube.com/watch?v=Acr-OoA7jew)
- [Innovating with AWS Confidential Computing: An Integrated Approach (CMP407)](https://www.youtube.com/watch?v=R2QxpJDEmY4)

## Summary

AWS markets the Nitro stack under its CC umbrella. AWS Nitro significantly reduces hypervisor attack surface by moving virtualization into hardware and minimizing privileged software. This gives strong operational security. However, AWS generally does not provide full TDX/SEV-SNP style memory encryption or confidential GPU VRAM isolation.

## Evaluation

AWS has two TEE offerings that are usable today
1. [Nitro Enclaves](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html)
2. [EC2 Instance Attestation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/nitrotpm-attestation.html)

Both of them are based on the Nitro TPM tools

### Nitro Enclaves

Nitro enclaves use the Nitro Hypervisor technology that offloads device virtualization to dedicated hardware on the host. They are processor agnostic and supported on most EC2 instance types. They come at no additional charge.

Nitro enclaves encapsulate a region of memory on an EC2 instance to only allow encrypted data in and out via a secure vsock channel. There is an attestation feature, which allows users to verify an enclave's identity and the code running inside it. The memory within the enclave is inaccessible by the hypervisor, and other VMs. The TCB is small by design.

Only [CPU workloads are supported](https://github.com/aws/aws-nitro-enclaves-cli/issues/543). You generally cannot:

- Attach NVIDIA GPU directly inside Nitro Enclave
- Run CUDA kernels inside enclave memory
- Do fully encrypted GPU memory execution like NVIDIA Confidential Computing

AWS Nitro Enclaves isolate CPU/memory resources, but GPU passthrough into the enclave is not the mainstream supported model. The enclave cannot guarantee PCIe data traffic is secure and has no mechanism to attest the state of a connected GPU.

The typical use case is a step in a workflow that needs to access confidential data.

#### Deployment

See [getting started](https://docs.aws.amazon.com/enclaves/latest/user/getting-started.html)

1. To launch an instance with nitro enclaves, set `--enclave-options 'Enabled=true'` in aws CLI, or `Nitro Enclave` to `Enable` in `Advanced details` in the EC2 launch GUI.
1. [Install Nitro CLI](https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave-cli-install.html) on the instance.
1. Build an enclave image file (e.g. from a Docker container)
2. Run and validate using tools in `nitro-cli`

### SEV-SNP on AWS

You can enable SEV-SNP memory encryption on select AWS instance types and AMIs. See [requirements](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-work-launch.html). To enable SEV-SNP, use `--cpu-options AmdSevSnp=enabled` in the AWS CLI, or `Advanced details - AMD SEV-SNP - Enabled`.

You can then verify that SEV-SNP is enabled by

```
ubuntu@ip-172-31-47-198:~$ sudo dmesg | grep -i -e rmp -e sev
[    3.207940] Memory Encryption Features active: AMD SEV SEV-ES SEV-SNP
[    3.209941] SEV: Status: SEV SEV-ES SEV-SNP
[    3.330965] SEV: APIC: wakeup_secondary_cpu() replaced with wakeup_cpu_via_vmgexit()
[    3.694232] SEV: Using SNP CPUID table, 38 entries present.
[    3.707953] SEV: SNP running at VMPL0.
[    4.904153] SEV: SNP guest platform devices initialized.
[   14.594861] sev-guest sev-guest: Initialized SEV guest driver (using VMPCK0 communication key)
```
There is a [tutorial for getting an attestation report from AMD](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/snp-attestation.html).

### EC Instance Attestation

EC2 Instance Attestation is a measured boot attestation system. With it, an user can

- Generate an attestable machine image (AMI)
- Get an attestation report with NitroTPM
- Launch the AMI as an EC2 instance and verify its state at boot

Attestation only verifies the guest booted correctly it does not isolate memory from the host at runtime.

EC2 instance attestation is available for Amazon Linux 2023. Its use case is for lift and shift of full applications. Unlike Nitro enclaves, it supports CUDA applications and GPUs. However, it gives less security guarantees than a Nitro enclave (or SNP-SEV/TDX type attestation). It does not guarantee

- runtime memory isolation from the hypervisor
- encrypted guest RAM inaccessible to the host
- GPU memory confidentiality
- protection against a malicious VMM/hypervisor

#### Deployment

1. Build an attestable AMI. See [tutorial](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/build-sample-ami.html)
2. Get a Nitro TPM attestation document. See [tutorial](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/attestation-get-doc.html)
3. Attest with a Key Management System, e.g. Amazon KMS. See [tutorial](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/prepare-attestation-service.html)

### Confidential Computing on AWS Elastic Kubernetes Service (EKS)

Confidential computing on EKS is provided by Nitro Enclaves (see above), or confidential containers. See [this tutorial](https://confidentialcontainers.org/docs/examples/aws-simple/) for confidential containers.