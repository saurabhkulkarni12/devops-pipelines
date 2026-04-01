pipeline {
    agent { label 'windows' }

    environment {
        TARGET_SERVERS = '192.168.9.110'
        IIS_SITE_PATH = 'C:\\inetpub\\wwwroot\\SimpleDotNetApp'
        STAGING_PATH   = 'D:\\Deployments\\Staging'
        BACKUP_PATH    = 'D:\\Deployments\\Backups'
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/jenkins-docs/simple-dotnet-web-app.git'
            }
        }

        stage('Build & Publish') {
            steps {
                powershell '''
                    dotnet restore
                    dotnet build ./simple-dotnet-web-app.sln -c Release
                    dotnet publish ./simple-dotnet-web-app.sln -c Release -o ./publish_output

                    Compress-Archive -Path "./publish_output/*" -DestinationPath "./release.zip" -Force
                '''
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