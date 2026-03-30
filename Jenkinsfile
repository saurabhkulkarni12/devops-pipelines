pipeline {
    agent any // Or specify a dedicated build node with .NET SDK installed
    environment {  
        TARGET_SERVERS = '192.168.9.110' 
        IIS_SITE_PATH = 'C:\\inetpub\\wwwroot\\SimpleDotNetApp'        STAGING_PATH = 'D:\\Deployments\\Staging'        BACKUP_PATH = 'D:\\Deployments\\Backups'    DEPLOY_CREDS = credentials('iis-deploy-creds') 
    }
    stages {        
        stage('Checkout Code') {
            steps {                // Jenkins automatically checks out the 'devops-pipelines' repo.// Now we explicitly checkout the application code.                checkout([$class: 'GitSCM', 
                    branches: [[name: '*/main']]
                    userRemoteConfigs: [url: 'https://github.com/saurabhkulkarni12/simple-dotnet-web-app.git']
                        }
        }

        stage('Build & Publish') {
            steps {
                powershell '''
                    # 1. Build and Publish the .NET App locally on Jenkins
                    dotnet publish ./simple-dotnet-web-app.sln -c Release -o ./publish_output
                    
                    # 2. Zip the artifact for easy transfer
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
                            # Authenticate for remote operations
                            \$secpasswd = ConvertTo-SecureString '${DEPLOY_CREDS_PSW}' -AsPlainText -Force
                            \$creds = New-Object System.Management.Automation.PSCredential ('${DEPLOY_CREDS_USR}', \$secpasswd)

                            # 1. Transfer the Zip to the Staging folder via SMB (Network Share)
                            Copy-Item -Path "./release.zip" -Destination "\\\\${server}\\D\$\\Deployments\\Staging\\release.zip" -Credential \$creds -Force

                            # 2. Execute Deployment Logic Remotely via WinRM
                            Invoke-Command -ComputerName ${server} -Credential \$creds -ScriptBlock {
                                param(\$SitePath, \$StagePath, \$BackupPath)

                                \$ZipFile = "\$StagePath\\release.zip"
                                \$Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
                                \$AppOffline = "\$SitePath\\app_offline.htm"

                                # Step A: Drop Maintenance Page
                                Set-Content -Path \$AppOffline -Value "<html><head><style>body{font-family:sans-serif;text-align:center;padding:50px;}</style></head><body><h1>System Maintenance</h1><p>We are upgrading the system. Please check back in a few minutes.</p></body></html>"
                                Start-Sleep -Seconds 3 # Wait for IIS to release locks

                                # Step B: Backup current live files
                                Compress-Archive -Path "\$SitePath\\*" -DestinationPath "\$BackupPath\\backup_\$Timestamp.zip" -Force

                                # Step C: Purge old files (Excluding the maintenance page!)
                                Get-ChildItem -Path \$SitePath -Exclude "app_offline.htm" | Remove-Item -Recurse -Force

                                # Step D: Extract new build
                                Expand-Archive -Path \$ZipFile -DestinationPath \$SitePath -Force

                                # Step E: Remove Maintenance Page to wake up IIS
                                Remove-Item -Path \$AppOffline -Force

                            } -ArgumentList "${IIS_SITE_PATH}", "${STAGING_PATH}", "${BACKUP_PATH}"
                        """
                    }
                }
            }
        }
    }
}
 