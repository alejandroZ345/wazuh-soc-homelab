# 🛡️ Cloud-Native Security Operations: Wazuh SIEM & XDR Deployment

📋 Project Overview
This project demonstrates the deployment and configuration of an enterprise-grade SIEM (Security Information and Event Management) and XDR (Extended Detection and Response) platform using Wazuh orchestrated through Docker Compose.
I am working to design this environment to simulate real-world security operations, focusing on threat detection, integrity monitoring, and regulatory compliance.

🎯 Key Objectives:
1. Infrastructure as Code (IaC): Streamline SIEM deployment using containerization for scalability and isolation.
2. Threat Detection: Implement active monitoring for brute-force attacks, unauthorized access, and malware patterns.
3. FIM (File Integrity Monitoring): Monitor critical system files for unauthorized changes in real-time.
4. Vulnerability Assessment: Automate the identification of CVEs across monitored endpoints.

🏗️ Technical Architecture
The environment is built on a Ryzen 7 architecture, leveraging Docker to maintain a lightweight yet powerful security stack:
1. Indexer (Wazuh Indexer): Highly scalable, full-text search and analytics engine.
2. Server (Wazuh Manager): The brain: analyzes data from agents and triggers alerts.
3. Dashboard (Wazuh Dashboard): Visualization and mining of security events.
4. Orchestation (Docker Compose): Managing the multi-container application lifecycle.

📊 Documentation & Knowledge Management
A core component of my security workflow is structured documentation. This project integrates principles of Knowledge Management, ensuring that every alert, configuration, and incident is documented for future auditability and team collaboration.

🛠️ Roadmap & Future Improvements
[ ] Integrate TheHive for advanced Incident Case Management.
[ ] Implement automated response scripts (Active Response).

👨‍💻 About the Author
Systems Engineer & Cybersecurity Professional
Certification: ISC2 Certified in Cybersecurity (CC)
Specialization: SecOps, Cloud Infrastructure, and Technical Documentation.
Portfolio: www.linkedin.com/in/alejandro-zavala-zenteno
