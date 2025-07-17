# Ephemeral Environments Workshop

<img width="1103" height="777" alt="Screenshot 2025-07-16 alle 10 29 31" src="https://github.com/user-attachments/assets/162a4149-0611-4afd-baec-be27f22ac415" />

<a href="https://graz-dev.github.io/ephemeral-environments-workshop/#/" target="_blank" style="display: inline-block; background: #23c664; color: #fff; font-weight: bold; padding: 0.3em 1em; border-radius: 16px; text-decoration: none; margin-bottom: 1em;">Live Workshop</a>

This repository contains the materials and slides for the **Ephemeral Environments Workshop**. The workshop is focused on demonstrating how to create, manage, and optimize cloud-native ephemeral environments using [Kubernetes](https://github.com/kubernetes/kubernetes), [Crossplane](https://github.com/crossplane/crossplane) and [kube-green](https://github.com/kube-green/kube-green).

## Workshop Topics

- Introduction to ephemeral environments and their benefits
- Overview of **Crossplane** and **kube-green** for cloud-native infrastructure
- Strategies for automating resource provisioning and sustainability
- Real-world use cases and best practices for development and testing

## Workshop Cases

- **Patch Kubernetes Operators**: 
  - Learn how to manage the shutdown and restart of a PostgreSQL cluster using the [CloudNativePG](https://cloudnative-pg.io/) operator orchestrated by Crossplane.Access the case on the [workshop slides](https://graz-dev.github.io/ephemeral-environments-workshop/#/9) or access the markdown guide in the `cases/cloud-native-pg/WORKSHOP.md` file.
    - Used in this workshop:
      - Kind
      - Helm
      - Crossplane
      - kube-green
      - No cloud provider subscription needed, it can be done fully on local environment
- **Patch AWS Infrastructure Resources**
  - Learn how to manage the availability of a *AWS S3 Bucket* using the [Crossplane AWS S3 Provider](https://marketplace.upbound.io/providers/upbound/provider-aws-s3/v1.23.1) and kube-green. Access the case on the [workshop slides](WIP) or access the markdown guide in the `cases/aws-s3/WORKSHOP.md` file.
    - Used in this workshop:
      - Kind
      - Helm
      - Crossplane
      - kube-green
      - *An AWS subscription is required for this workshop*

