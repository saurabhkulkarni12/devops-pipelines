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

        stage('Deploy to IIS Servers') {
            steps {
                script {
                    def servers = env.TARGET_SERVERS.split(',')

                    withCredentials([usernamePassword(
                        credentialsId: 'iis-deploy-creds',
                        usernameVariable: 'USERNAME',
                        passwordVariable: 'PASSWORD'
                    )]) {

                        for (server in servers) {
                            echo "Deploying to ${server}"

                            // pass server as env var to avoid Groovy interpolation
                            withEnv(["TARGET_SERVER=${server}"]) {

                                powershell '''
                                    $server = $env:TARGET_SERVER
                                    $username = $env:USERNAME
                                    $password = $env:PASSWORD

                                    # Build admin share path safely
                                    $drive = "\\\\" + $server + "\\D$"

                                    # Map drive
                                    net use $drive /user:$username $password

                                    # Copy artifact
                                    Copy-Item -Path ".\\release.zip" -Destination "$drive\\Deployments\\Staging\\release.zip" -Force

                                    # Disconnect
                                    net use $drive /delete

                                    # Prepare credentials
                                    $secpasswd = ConvertTo-SecureString $password -AsPlainText -Force
                                    $creds = New-Object System.Management.Automation.PSCredential ($username, $secpasswd)

                                    Invoke-Command -ComputerName $server -Credential $creds -ScriptBlock {
                                        param($SitePath, $StagePath, $BackupPath)

                                        $ZipFile   = "$StagePath\\release.zip"
                                        $Timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
                                        $AppOffline = "$SitePath\\app_offline.htm"

                                        # Maintenance mode
                                        Set-Content -Path $AppOffline -Value "<html><body><h1>Maintenance</h1></body></html>"
                                        Start-Sleep -Seconds 3

                                        # Backup
                                        Compress-Archive -Path "$SitePath\\*" -DestinationPath "$BackupPath\\backup_$Timestamp.zip" -Force

                                        # Cleanup
                                        Get-ChildItem -Path $SitePath -Exclude "app_offline.htm" | Remove-Item -Recurse -Force

                                        # Deploy
                                        Expand-Archive -Path $ZipFile -DestinationPath $SitePath -Force

                                        # Bring app online
                                        Remove-Item -Path $AppOffline -Force
                                    } -ArgumentList $env:IIS_SITE_PATH, $env:STAGING_PATH, $env:BACKUP_PATH
                                '''
                            }
                        }
                    }
                }
            }
        }
    }
}