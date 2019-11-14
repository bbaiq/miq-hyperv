# win-dsc

# Prérequis : 
- Powershell v5 obligatoire : Lancer le script d'upgrade v5

```
$url = "https://raw.githubusercontent.com/jborean93/ansible-windows/master/scripts/Upgrade-PowerShell.ps1"
$file = "$env:temp\Upgrade-PowerShell.ps1"
$username = "Administrator"
$password = "Password"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Force

# version can be 3.0, 4.0 or 5.1
&$file -Version 5.1 -Username $username -Password $password -Verbose
```



- Vérifier version Powershell via $PsVersionTable.
- Machines dans le domaine HBGT (variable group_vars)

# Autoconfiguration ConfigureRemotingForAnsible.ps1
Le script ConfigureRemotingForAnsible.ps1 automatise :

- La création de 2 listeners HTTP et HTTPS (tcp/5985 et tcp/5986). Le listener HTTPS est inscrit avec un certificat self signé avec CN = HOSTNAME.
- Le certifcat a une durée de validité = valeure passée dans la cde certValidityDays 365 
- ForceNewSSLCert pour réinscrire le certificat self-signé qui a expiré par exemple
- Active l'authentification basic
- Possible d'ajouter une authentification CredSSP avec le paramètre -EnableCredSSP(necessite des modules credssp supplementaires sur la vm ansible)

```
pip install --upgrade pip
python3 -m pip install --user --upgrade pyopenssl
python3 -m pip install --user --upgrade pywinrm[credssp]
python3 -m pip install --user --upgrade requests-credssp
```

- Ajoute et active 2 règles de parefeu IN pour winrRM TCP 5985 et TCP 5986

# Paramètres du script d'autoconfiguration
Lancer le script d'autoconfiguration WinRM pour ansible avec un powershell en tant que Administrateur

```
$url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
$file = "$env:temp\ConfigureRemotingForAnsible.ps1"

(New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)

powershell.exe -ExecutionPolicy ByPass -File $file -Verbose -certValidityDays 365 -ForceNewSSLCert
```

# Check de la configuration winRM
Pour voir la configuration actuelle du service

```
winrm get winrm/config/Service
```

Pour voir les listeners de winRM

```
winrm enumerate winrm/config/Listener
```

Pour remove tous les listener winRM

```
Remove-Item -Path WSMan:\localhost\Listener\* -Recurse -Force
```

Pour remove listeners that are run over HTTPS

```
Get-ChildItem -Path WSMan:\localhost\Listener | Where-Object { $_.Keys -contains "Transport=HTTPS" } | Remove-Item -Recurse -Force
```


# Playbook telegraf.yml
- Ajouter les hosts dans env/inventory.ini
- Le playbook telegraf.yml recheche si une version telegraf est existante
- Si existante, le playbook arrete le service
- Il copie ensuite les fichiers telegraf.exe et telegrfa.conf dans C:\program Files\telegraf
- Il inscrit le service telegraf puis démarre le service
- Le fichier de conf telegraf pousse les métriques en HTTPS vers la bdD temporelles influxDB
- 1 datasource influxdb interroge les métriques et construit le dashboard Grafana


