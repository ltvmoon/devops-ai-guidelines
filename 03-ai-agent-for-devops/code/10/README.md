# Chapter 10: Building a Complex Agent with Actions - Real AWS Production Scenario

A production-ready AI incident response agent that detects database connection exhaustion across multiple backend pods, reboots AWS RDS instances, and notifies teams via Slack.

## What's New in Chapter 10

In Chapter 9, we added decision-making with a single placeholder action (pod restart). In Chapter 10, we build a complete incident response workflow for a real-world AWS scenario:

- **Real Scenario**: Three-tier AWS app with Java backend on EKS hitting RDS "Too many connections"
- **AWS Actions**: Reboot RDS instances using boto3 (real or simulated)
- **Slack Notifications**: Send incident reports to team channels
- **Action Chaining**: Read logs → Detect issue → Reboot RDS → Notify Slack → Report
- **Multi-Pod Analysis**: Correlate errors across 3 backend pod log files
- **Dual Mode**: Works with real AWS/Slack credentials or in placeholder mode for learning

## Features

- 📊 **Multi-Pod Log Analysis**: Analyze logs from multiple backend pods simultaneously
- 🗄️ **RDS Management**: Reboot AWS RDS database instances to reset connections
- 💬 **Slack Integration**: Send detailed incident notifications to team channels
- 🔗 **Action Chaining**: Execute multiple actions in sequence after single approval
- ☁️ **AWS Integration**: Real boto3 integration or placeholder mode
- 🔒 **Safety First**: All destructive actions require explicit user approval

## Architecture

```
Three-Tier Application on AWS:
┌─────────────┐    ┌──────────────────────┐    ┌─────────────┐
│  CloudFront  │───▶│  EKS Backend Pods    │───▶│  RDS MySQL  │
│  + S3        │    │  (Java Spring Boot)  │    │  (orders-db)│
└─────────────┘    │  Pod 1: 50 conns     │    └─────────────┘
                   │  Pod 2: 50 conns     │    ┌─────────────┐
                   │  Pod 3: 50 conns     │───▶│  ElastiCache│
                   └──────────────────────┘    │  (Redis)    │
                          │                     └─────────────┘
                          ▼
                   Max connections: 150
                   3 pods × 50 = 150 (exhausted!)
```

## Project Structure

```
10/
├── app.py                    # Streamlit application
├── system_prompt.txt         # AI agent instructions (AWS-focused)
├── src/
│   ├── __init__.py
│   ├── config.py            # Configuration (Gemini + AWS + Slack)
│   ├── models/
│   │   └── gemini.py
│   ├── tools/
│   │   ├── log_reader.py    # Log reading tools
│   │   ├── actions.py       # Kubernetes pod restart
│   │   ├── aws_actions.py   # NEW: RDS reboot (boto3)
│   │   └── slack_notifier.py # NEW: Slack webhook
│   ├── agents/
│   │   └── log_analyzer.py  # Agent with action chaining
│   └── utils/
│       └── response.py
├── logs/                     # Multi-pod log files
│   ├── backend-orders-pod1.log
│   ├── backend-orders-pod2.log
│   └── backend-orders-pod3.log
├── requirements.txt
├── .env.example
└── Makefile
```

## Setup

1. **Create virtual environment:**
```bash
python3 -m venv venv
source venv/bin/activate  # or: conda activate ai-agent
```

2. **Install dependencies:**
```bash
pip install -r requirements.txt
```

3. **Set up environment:**
```bash
cp .env.example .env
# Edit .env and add your GEMINI_API_KEY
```

## Usage

**Run the Streamlit app:**
```bash
streamlit run app.py
```

Or use the Makefile:
```bash
make run
```

The app will open in your browser at `http://localhost:8501`

### (Optional) Configure AWS and Slack

The agent works in **placeholder mode** by default (no real AWS/Slack needed):

```bash
# To enable real AWS actions:
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_REGION=us-east-1

# To enable real Slack notifications:
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
SLACK_CHANNEL=#devops-alerts
```

## How to Use

1. **Start the app** - Run `streamlit run app.py`
2. **Ask about logs** - e.g., "Check all logs for database issues"
3. **Review analysis** - Agent reads all pod logs and correlates errors
4. **Approve actions** - Agent asks before rebooting RDS or sending Slack messages
5. **Get next steps** - Agent provides monitoring and prevention guidance

### Example Conversation

```
You: Check all logs for database issues

AI: [Reads all 3 pod log files, correlates errors]

    Issue Summary: RDS database connection exhaustion across all backend pods

    Severity: P1

    Analysis:
    - Affected systems: RDS instance orders-db-prod, 3 backend pods
    - Root cause: 3 pods × 50 max connections = 150, exhausting RDS max_connections (150)
    - Impact: Order processing failing, payment verification failing, health checks degraded
    - Evidence: "Too many connections" errors starting at 10:12:45 across all pods

    Recommended Actions:
    1. Reboot RDS instance 'orders-db-prod' to reset all connections
    2. Send incident notification to Slack #devops-alerts

    Would you like me to proceed? (yes/no)

You: yes

AI: [Reboots RDS, sends Slack notification]

    I have rebooted RDS instance orders-db-prod and notified the team on Slack.

    Next Steps:
    - Monitor RDS instance status and connection counts
    - Investigate why 3 pods × 50 connections exceeded the limit
    - Consider: reduce per-pod pool size to 40, or enable RDS Proxy for connection pooling
```

## Dual Mode Operation

| Mode | When | What Happens |
|------|------|-------------|
| **Placeholder** | No AWS/Slack credentials | Actions print what they would do |
| **Real** | Credentials configured | Actions execute against real AWS/Slack |

Both modes provide identical agent behavior — the only difference is whether actions are simulated or real.

## Tools Available

| Tool | Type | Description |
|------|------|-------------|
| `read_log_file` | Auto | Read a specific log file |
| `list_log_files` | Auto | List all available log files |
| `search_logs` | Auto | Search for patterns in logs |
| `restart_kubernetes_pod` | Approval | Restart a failed K8s pod |
| `reboot_rds_instance` | Approval | Reboot an AWS RDS database |
| `send_slack_notification` | Approval | Send incident report to Slack |

## Troubleshooting

**Port already in use:**
```bash
streamlit run app.py --server.port 8502
```

**API key not found:**
- Check `.env` file exists
- Verify `GEMINI_API_KEY` is set
- Restart the Streamlit app

**Logs directory missing:**
- Create `logs/` directory
- Add sample log files
- Restart the app
