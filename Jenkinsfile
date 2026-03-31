pipeline {
    agent any

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
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/jenkins-docs/simple-dotnet-web-app.git'
                    ]]
                ])
            }
        }

        stage('Build & Publish') {
            steps {
                powershell '''
                    # Build and publish .NET app
                    dotnet publish ./simple-dotnet-web-app.sln -c Release -o ./publish_output

                    # Zip artifact
                    Compress-Archive -Path "./publish_output/*" -DestinationPath "./release.zip" -Force
                '''
            }
        }

        stage('Deploy to IIS Servers') {
            steps {
                script {
                    def servers = env.TARGET_SERVERS.split(',')

                    for (server in servers) {
                        echo "Deploying to ${server}..."

                        powershell """
                            \$secpasswd = ConvertTo-SecureString '${DEPLOY_CREDS_PSW}' -AsPlainText -Force
                            \$creds = New-Object System.Management.Automation.PSCredential ('${DEPLOY_CREDS_USR}', \$secpasswd)

                            # Copy artifact to remote server
                            Copy-Item -Path "./release.zip" -Destination "\\\\${server}\\D\$\\Deployments\\Staging\\release.zip" -Credential \$creds -Force

                            # Remote deployment
                            Invoke-Command -ComputerName ${server} -Credential \$creds -ScriptBlock {
                                param(\$SitePath, \$StagePath, \$BackupPath)

                                \$ZipFile   = "\$StagePath\\release.zip"
                                \$Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
                                \$AppOffline = "\$SitePath\\app_offline.htm"

                                # Maintenance mode
                                Set-Content -Path \$AppOffline -Value "<html><body><h1>Maintenance</h1></body></html>"
                                Start-Sleep -Seconds 3

                                # Backup
                                Compress-Archive -Path "\$SitePath\\*" -DestinationPath "\$BackupPath\\backup_\$Timestamp.zip" -Force

                                # Cleanup (except maintenance file)
                                Get-ChildItem -Path \$SitePath -Exclude "app_offline.htm" | Remove-Item -Recurse -Force

                                # Deploy new build
                                Expand-Archive -Path \$ZipFile -DestinationPath \$SitePath -Force

                                # Bring app online
                                Remove-Item -Path \$AppOffline -Force
                            } -ArgumentList "${IIS_SITE_PATH}", "${STAGING_PATH}", "${BACKUP_PATH}"
                        """
                    }
                }
            }
        }
    }
}