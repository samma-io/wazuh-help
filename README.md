# My Wazuh Toolkit

This repository contains a collection of personal scripts, configurations, and actions I use to customize and enhance my [Wazuh](https://wazuh.com/) instances. Wazuh is a powerful open-source Security Information and Event Management (SIEM) and Extended Detection and Response (XDR) platform, and these tools are designed to make it work even better for my specific needs.

## Repository Structure

The repository is organized into three main categories, each in its own directory:

1.  **🚀 Pre-Production Tasks:** Essential configurations and hardening steps to perform *before* deploying a Wazuh instance into a live production environment.

2.  **🛠️ Operational Improvements:** Scripts and guides for enhancing existing Wazuh functionality. This includes tasks like adding better monitoring on specific log sources or integrating alerts with external ticketing systems.

3.  **🧩 Functionality Extensions:** Code and instructions for extending Wazuh's capabilities by integrating new, external data sources. The goal is to leverage the power of Wazuh's correlation engine for more security insights from platforms like Detectify, Secureframe, and more.

## How to Use This Repository

Each folder corresponds to one of the categories listed above. Browse the directories to find the specific scripts, configuration files, and documentation relevant to your needs.

---

### 🚀 1. Pre-Production Tasks
This section contains all the necessary checks and configurations I perform before deploying Wazuh into production.

### 🛠️ 2. Operational Improvements
Here you will find ways to improve the day-to-day functionality of an existing Wazuh deployment. This includes things like adding better monitoring on specific log sources or sending alerts to ticketing systems like Jira or TheHive.

### 🧩 3. Functionality Extensions
This is where I explore extending the capabilities of Wazuh beyond its standard feature set by integrating new tools and data sources.

## ⚠️ Disclaimer

The configurations and scripts in this repository are from my personal working environment and are provided as-is. **Always test them thoroughly in a non-production environment before applying them to your live systems.**
