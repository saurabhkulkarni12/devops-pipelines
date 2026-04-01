pipeline {
    agent { label 'windows-agent' }

    environment {
        TARGET_SERVERS = '192.168.9.110'
        IIS_SITE_PATH = 'C:\\inetpub\\wwwroot\\SimpleDotNetApp'
        STAGING_PATH   = 'D:\\Deployments\\Staging'
        BACKUP_PATH    = 'D:\\Deployments\\Backups'
        DEPLOY_CREDS   = credentials('iis-deploy-creds')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/saurabhkulkarni12/simple-dotnet-web-app.git'
            }
        }

        stage('Build & Publish') {
            steps {
                powershell '''
                    dotnet publish ./simple-dotnet-web-app.sln -c Release -o ./publish_output
                    Compress-Archive -Path "./publish_output/*" -DestinationPath "./release.zip" -Force
                '''
            }
        }

        stage('Deploy to IIS Servers') {
            steps {
                script {
                    def servers = env.TARGET_SERVERS.split(',')

                    for (server in servers) {
                        powershell """
                            \$secpasswd = ConvertTo-SecureString '${DEPLOY_CREDS_PSW}' -AsPlainText -Force
                            \$creds = New-Object System.Management.Automation.PSCredential ('${DEPLOY_CREDS_USR}', \$secpasswd)

                            Copy-Item -Path "./release.zip" -Destination "\\\\${server}\\D\$\\Deployments\\Staging\\release.zip" -Credential \$creds -Force

                            Invoke-Command -ComputerName ${server} -Credential \$creds -ScriptBlock {
                                param(\$SitePath, \$StagePath, \$BackupPath)

                                \$ZipFile   = "\$StagePath\\release.zip"
                                \$Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
                                \$AppOffline = "\$SitePath\\app_offline.htm"

                                Set-Content -Path \$AppOffline -Value "<h1>Maintenance</h1>"
                                Start-Sleep -Seconds 3

                                Compress-Archive -Path "\$SitePath\\*" -DestinationPath "\$BackupPath\\backup_\$Timestamp.zip" -Force

                                Get-ChildItem -Path \$SitePath -Exclude "app_offline.htm" | Remove-Item -Recurse -Force

                                Expand-Archive -Path \$ZipFile -DestinationPath \$SitePath -Force

                                Remove-Item -Path \$AppOffline -Force
                            } -ArgumentList "${IIS_SITE_PATH}", "${STAGING_PATH}", "${BACKUP_PATH}"
                        """
                    }
                }
            }
        }
    }
}