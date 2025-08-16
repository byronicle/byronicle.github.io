---
layout: post
title: "Condor Enphase"
---
## The Journey Begins

After being laid off from Verily, I decided to turn a challenge into an opportunity. Rather than just practicing programming skills with theoretical projects, I wanted to build something practical that would solve a real problem in my home. That's when I looked at my Enphase solar system and realized I could create a better way to monitor and visualize its performance.

## The Problem

While Enphase provides its own monitoring tools, I wanted more control over my data and the ability to create custom visualizations that matched my specific needs. Plus, this would be the perfect project to sharpen my skills in modern infrastructure and DevOps practices.

## Version 1: The Raspberry Pi Solution

### Architecture Overview

My first iteration runs entirely on a Raspberry Pi, creating a self-contained monitoring system that operates within my home network. Here's how it works:

1. **Data Collection**: A Python application polls the local Enphase Envoy API to fetch real-time solar production and consumption metrics
2. **Data Storage**: Time-series data is stored in InfluxDB v2 for efficient querying and long-term retention
3. **Visualization**: Grafana provides customizable dashboards to display the metrics in meaningful ways
4. **Containerization**: All components run as Docker containers orchestrated by Docker Compose
5. **Remote Access**: Tailscale enables secure access to the Grafana dashboard from anywhere

### Technical Implementation

The core of the system is a Python class that handles the complexity of Enphase's token-based authentication and API interactions. I leveraged ChatGPT to help structure the initial code, then refined it based on the specific quirks of the Envoy API.

The deployment pipeline showcases modern CI/CD practices:
- GitHub Actions automatically build and deploy updates
- A self-hosted runner on the Raspberry Pi pulls and runs the latest container images
- Docker Compose ensures all services start correctly and can recover from failures

### Key Features

- **Real-time Monitoring**: Polls data every few minutes for up-to-date information
- **Historical Analysis**: InfluxDB retention policies preserve data for long-term trend analysis
- **Custom Dashboards**: Grafana allows creation of personalized views (daily production, consumption patterns, net metering, etc.)
- **Secure Remote Access**: Tailscale VPN means no exposed ports or complex firewall rules

## Version 2: Scaling to the Cloud

Currently, I'm experimenting with cloud deployment options to explore different infrastructure patterns:

### Google Compute Engine
Running the entire stack on a single VM instance, similar to the Raspberry Pi setup but with cloud reliability and scaling potential.

### Google Kubernetes Engine (GKE)
Decomposing the application into microservices running on Kubernetes, allowing for:
- Better resource utilization
- Automatic scaling based on load
- Rolling updates with zero downtime
- Enhanced monitoring and observability

### Infrastructure as Code
Using Terraform to provision all Google Cloud resources ensures:
- Reproducible deployments
- Version-controlled infrastructure
- Easy environment replication (dev/staging/prod)

## Lessons Learned

1. **Start Simple**: The Raspberry Pi version proved the concept before investing in cloud infrastructure
2. **Containerization Early**: Docker made the transition from local to cloud seamless
3. **Authentication Complexity**: Enphase's token-based auth required careful handling and secure storage
4. **Time-Series Data**: InfluxDB's purpose-built design significantly outperforms traditional databases for this use case
5. **GitOps Workflow**: Automated deployments via GitHub Actions reduced manual intervention and potential errors

## Technical Stack

- **Languages**: Python 3.11+
- **Data Storage**: InfluxDB v2
- **Visualization**: Grafana
- **Container Orchestration**: Docker Compose (local), Kubernetes (cloud)
- **Infrastructure**: Terraform for GCP resources
- **CI/CD**: GitHub Actions with self-hosted runners
- **Networking**: Tailscale for secure remote access
- **Cloud Platform**: Google Cloud Platform (Compute Engine, GKE, Secret Manager)

## What's Next?

- Implementing predictive analytics using historical data
- Adding weather correlation to understand production patterns
- Creating mobile-friendly dashboard views
- Exploring cost optimization strategies for cloud deployment
- Adding alerting for system anomalies or production issues

## Open Source

The entire project is open source and available on GitHub. Feel free to fork it for your own solar monitoring needs or contribute improvements!

**Repository**: [github.com/byronicle/condor-enphase](https://github.com/byronicle/condor-enphase)

âš ï¸ **Note**: The project is under active development, so expect breaking changes as I continue to refine the architecture and add features.

## Conclusion

This project transformed from a simple learning exercise into a practical tool that provides real value. It demonstrates how modern DevOps practices can be applied to home automation projects, and how starting with a minimal viable product (Raspberry Pi) can evolve into a cloud-native solution.

Whether you're interested in solar monitoring, learning DevOps, or just enjoy tinkering with home automation, I hope this project provides inspiration or useful code for your own endeavors.