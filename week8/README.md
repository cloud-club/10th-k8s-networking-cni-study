# Week 8. Kubernetes Networking Common Issue Troubleshooting & Operations

운영에서 자주 만나는 네트워크 장애를 분석하는 주차입니다. Pod, Service, External 통신 문제를 분해해서 보고, CNI와 kube-proxy, DNS, IPAM 관련 이슈를 점검합니다.

## 학습 포인트

- Pod-to-Pod / Pod-to-Service / External 통신 장애 분석
- CNI 장애 Troubleshooting
- IPAM / IP Pool 문제 분석
- Routing / NetworkPolicy / DNS / kube-proxy 문제 분석

## 정리할 내용

- 장애를 어디서부터 분해해서 볼지
- 네트워크 문제를 확인하는 기본 점검 순서
- 운영 중 자주 터지는 케이스와 대응 방법
