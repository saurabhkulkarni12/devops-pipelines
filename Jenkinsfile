pipeline {
    agent any

    environment {
        TARGET_SERVER = '192.168.9.110'
        IIS_SITE_PATH = 'C:\\inetpub\\wwwroot\\SimpleDotNetApp'
        BACKUP_PATH   = 'D:\\Deployments\\Backups'
        BUILD_PATH    = 'D:\\BuildOutput '   // Adjust based on your build
        DEPLOY_CREDS  = credentials('iis-deploy-creds')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git 'https://github.com/saurabhkulkarni12/simple-dotnet-web-app.git'
            }
        }

        stage('Build Application') {
            steps {
                bat 'dotnet publish -c Release -o %BUILD_PATH%'
            }
        }

        stage('Deploy to IIS Server') {
            steps {
                powershell '''
                $secpasswd = ConvertTo-SecureString "$env:DEPLOY_CREDS_PSW" -AsPlainText -Force
                $cred = New-Object System.Management.Automation.PSCredential ("$env:DEPLOY_CREDS_USR", $secpasswd)

                $session = New-PSSession -ComputerName $env:TARGET_SERVER -Credential $cred

                # Stop IIS
                Invoke-Command -Session $session -ScriptBlock {
                    Stop-Service W3SVC -Force
                }

                # Take Backup
                Invoke-Command -Session $session -ScriptBlock {
                    $timestamp = Get-Date -Format "yyyyMMddHHmmss"
                    Copy-Item "$env:IIS_SITE_PATH" "$env:BACKUP_PATH\\Backup_$timestamp" -Recurse
                }

                # Clean old files
                Invoke-Command -Session $session -ScriptBlock {
                    Remove-Item "$env:IIS_SITE_PATH\\*" -Recurse -Force
                }

                # Copy new build
                Copy-Item "$env:BUILD_PATH\\*" -Destination "$env:IIS_SITE_PATH" -Recurse -ToSession $session

                # Start IIS
                Invoke-Command -Session $session -ScriptBlock {
                    Start-Service W3SVC
                }

                # Verify IIS
                Invoke-Command -Session $session -ScriptBlock {
                    Get-Service W3SVC
                }

                Remove-PSSession $session
                '''
            }
        }
    }

    post {
        success {
            echo '✅ Deployment Successful!'
        }
        failure {
            echo '❌ Deployment Failed!'
        }
    }
}