# Flask Application CI/CD with GitHub Actions and EC2

## Table of Contents
1. [Introduction](#introduction)
2. [Prerequisites](#prerequisites)
3. [Repository Setup](#repository-setup)
4. [EC2 Instance Setup](#ec2-instance-setup)
5. [GitHub Secrets Configuration](#github-secrets-configuration)
6. [GitHub Actions Workflow](#github-actions-workflow)
7. [Workflow Explanation](#workflow-explanation)
8. [Troubleshooting](#troubleshooting)
9. [Conclusion](#conclusion)

## Introduction

This documentation outlines the process of implementing Continuous Integration and Continuous Deployment (CI/CD) for a Flask application using GitHub Actions and an Amazon EC2 instance. The setup automates testing and deployment processes, ensuring code quality and streamlined releases.

## Prerequisites

- A GitHub account and repository containing your Flask application
- An Amazon Web Services (AWS) account
- Basic knowledge of Python, Flask, Git, and Linux commands

## Repository Setup

Ensure your repository has the following structure:

```
your-repo/
├── .github/
│   └── workflows/
│       └── test.yaml
├── tests/
│   └── test_app.py
├── app.py
├── requirements.txt
└── README.md
```

## EC2 Instance Setup

1. Launch an Ubuntu EC2 instance in your AWS account.
2. Configure security groups to allow inbound traffic on port 22 (SSH) and the port your Flask app uses (typically 5000).
3. Create or use an existing key pair for SSH access.

## GitHub Secrets Configuration

Add the following secrets to your GitHub repository:

1. `SSH_PRIVATE_KEY`: The content of your EC2 instance's private key file (.pem)
2. `HOST`: Your EC2 instance's public IP or DNS
3. `USERNAME`: The username for SSH access (usually 'ubuntu' for Ubuntu EC2 instances)
4. `KNOWN_HOSTS`: Output of `ssh-keyscan -H YOUR_EC2_IP`

To add secrets:
1. Go to your GitHub repository
2. Click "Settings" > "Secrets and variables" > "Actions"
3. Click "New repository secret" for each secret

## GitHub Actions Workflow

Create a file named `test.yaml` in the `.github/workflows/` directory with the following content:

```yaml
name: Python Flask CI/CD

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.9'
    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt
    - name: Run tests
      run: |
        python -m unittest discover tests

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v3
    
    - name: Install SSH Key
      uses: shimataro/ssh-key-action@v2
      with:
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        known_hosts: ${{ secrets.KNOWN_HOSTS }}
    
    - name: Deploy to Staging
      env:
        HOST: ${{ secrets.HOST }}
        USERNAME: ${{ secrets.USERNAME }}
      run: |
        ssh $USERNAME@$HOST << EOF
          # Update system
          sudo apt update && sudo apt upgrade -y

          # Ensure required packages are installed
          sudo apt install -y python3-venv

          # Set up application directory
          mkdir -p ~/flask-app
          cd ~/flask-app

          # Set up Git repository
          git init
          git remote add origin https://github.com/${{ github.repository }}.git
          git fetch origin main
          git reset --hard origin/main

          # Set up Python virtual environment
          python3 -m venv venv
          source venv/bin/activate
          pip install -r requirements.txt

          # Create systemd service file
          sudo tee /etc/systemd/system/flask-app.service > /dev/null << EOT
          [Unit]
          Description=Flask App
          After=network.target

          [Service]
          User=$USERNAME
          WorkingDirectory=/home/$USERNAME/flask-app
          ExecStart=/home/$USERNAME/flask-app/venv/bin/python app.py
          Restart=always

          [Install]
          WantedBy=multi-user.target
          EOT

          # Reload systemd, enable and restart the service
          sudo systemctl daemon-reload
          sudo systemctl enable flask-app.service
          sudo systemctl restart flask-app.service

          # Output service status for logging
          sudo systemctl status flask-app.service
        
```

## Workflow Explanation

The workflow consists of two jobs:

1. **test**: Runs on every push and pull request.
   - Sets up Python
   - Installs dependencies
   - Runs unit tests

2. **deploy-staging**: Runs only on pushes to the main branch, after tests pass.
   - Sets up SSH access to the EC2 instance
   - Updates the system
   - Sets up the application directory
   - Initializes and updates the Git repository
   - Creates and activates a Python virtual environment
   - Installs dependencies
   - Creates a systemd service for the Flask app
   - Starts the Flask app service

## Troubleshooting

Common issues and solutions:

1. **SSH Connection Fails**: Ensure the EC2 instance's security group allows inbound traffic on port 22 and that the SSH key is correctly set up in GitHub secrets.

2. **Python Package Installation Fails**: Check if `requirements.txt` is up-to-date and all packages are compatible with the Python version specified in the workflow.

3. **Flask App Doesn't Start**: Verify that `app.py` is in the root directory of your repository and contains the necessary code to run the Flask app.

4. **Changes Not Reflected After Deployment**: Ensure that the GitHub Actions workflow is triggered on pushes to the main branch and that the deployment job completes successfully.

## Conclusion

This CI/CD setup automates the testing and deployment of your Flask application, improving code quality and deployment efficiency. Regular monitoring of GitHub Actions logs and EC2 instance health is recommended to ensure smooth operation.

For further customization or advanced use cases, refer to the official documentation for [GitHub Actions](https://docs.github.com/en/actions) and [AWS EC2](https://docs.aws.amazon.com/ec2/).
