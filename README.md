1. Tartományvezérlő előléptetés (AD DS)
A szerver már telepítve van, csak elő kell léptetni.
Server Managerben: Notifications (zászló ikon) → Promote this server to a domain controller
Wizard lépések:
1. Add a new forest
   Root domain name: vizsga25.jedlik

2. Domain Controller Options
   → Minden alapértelmezett marad
   → DSRM jelszó: pl. Password123!

3. DNS Options → Next (figyelmeztetés ignorálható)

4. Additional Options → NetBIOS: VIZSGA25 (auto)

5. Paths → alapértelmezett

6. Review → Install
   (Gép újraindul automatikusan)

Képernyőkép kell: AD DS szerepkör a Server Managerben, az előléptetés varázslóból az erdőnév megadása.


2. Szervezeti egységek (OU) létrehozása
Active Directory Users and Computers (ADUC) → jobb klikk a domainre → New → Organizational Unit
Létrehozandó struktúra:

vizsga25.jedlik
├── Hallgatók
├── Titkárság
└── Tanárok
    ├── Humán
    └── Reál
Sorrend:

Hallgatók → jobb klikk a domainre → New → OU
Titkárság → ugyanígy
Tanárok → ugyanígy
Humán → jobb klikk a Tanárok OU-ra → New → OU
Reál → jobb klikk a Tanárok OU-ra → New → OU


Képernyőkép kell: a kész OU-fa az ADUC-ban.


3. Felhasználók és csoportok
Felhasználók létrehozása – jobb klikk a megfelelő OU-ra → New → User
MezőÉrtékFirst nameErősLast nameElekFull nameErős ElekUser logon nameeros.elekOUHallgatók
Jelszó beállításnál: "User must change password at next logon" → NE pipáld be!
Ugyanígy a többi felhasználónak:
Teljes névBejelentkezési névOUFarkas Leofarkas.leoTanárok/HumánVezetékes Valérvezetekes.valerTitkárság
Csoportok létrehozása – jobb klikk a megfelelő OU-ra → New → Group
CsoportnévHatókörTípusOUIgazgatóságDomain LocalSecurityTanárokNappali képzésGlobalSecurityHallgatókEsti képzésGlobalSecurityHallgatókEmailDomain LocalDistributionTitkárság
Csoporttagságok beállítása – jobb klikk a felhasználón → Properties → Member Of → Add
FelhasználóCsoportErős ElekNappali képzésVezetékes ValérIgazgatóságIgazgatóság, Nappali képzés, Esti képzés, EmailEmail
Az utolsó sorhoz: jobb klikk az Email csoporton → Properties → Members → Add → mind a 4 csoportot add hozzá.

Képernyőkép kell: OU-k, csoportok típusai, csoportok tagjai.


4. PowerShell parancsok
powershell# Rendszergazdák OU létrehozása a gyökér AD-be
New-ADOrganizationalUnit -Name "Rendszergazdák" -Path "DC=vizsga25,DC=jedlik"

# adminka felhasználó létrehozása
New-ADUser `
  -Name "adminka" `
  -GivenName "adminka" `
  -SamAccountName "adminka" `
  -UserPrincipalName "adminka@vizsga25.jedlik" `
  -Path "OU=Rendszergazdák,DC=vizsga25,DC=jedlik" `
  -AccountPassword (ConvertTo-SecureString "Password123!" -AsPlainText -Force) `
  -Enabled $true `
  -PasswordNeverExpires $true

# adminok globális biztonsági csoport létrehozása
New-ADGroup `
  -Name "adminok" `
  -GroupScope Global `
  -GroupCategory Security `
  -Path "OU=Rendszergazdák,DC=vizsga25,DC=jedlik"

# adminka hozzáadása az adminok csoporthoz
Add-ADGroupMember -Identity "adminok" -Members "adminka"

# IIS webszerver szerepkör telepítése
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

Képernyőkép kell: minden parancs és kimenete.


5. DHCP beállítás
Server Manager → Tools → DHCP
Jobb klikk az IPv4-re → New Scope

Scope neve:       tetszőleges (pl. Iroda)
IP tartomány:     192.168.25.0/24
  Start IP:       192.168.25.120
  End IP:         192.168.25.180
  Subnet mask:    255.255.255.0

Kizárások:        (nem kötelező, ha nincs)

Default Gateway:  192.168.25.1
DNS szerver:      192.168.25.10

Activate scope:   Yes (varázsló végén)

Képernyőkép kell: az aktív hatókör beállításai (jobb klikk → Properties).


6. Csoportházirend (GPO)
Group Policy Management (gpmc.msc)
1. GPO létrehozása és összekapcsolása:
   Jobb klikk a "Hallgatók" OU-ra
   → Create a GPO in this domain, and Link it here
   → Név: Tanulók_szabályozása
Szoftvertelepítés (xmlnotepad):
GPO jobb klikk → Edit
Computer Configuration → Policies →
Software Settings → Software Installation
→ New → Package → tallózd a gyökérkönyvtárból az xmlnotepad.msi fájlt
→ Deployed
Háttérkép tiltása:
User Configuration → Policies →
Administrative Templates → Control Panel → Personalization
→ "Prevent changing desktop background" → Enabled
Home könyvtár beállítása:
User Configuration → Policies →
Windows Settings → Folder Redirection → (nem ez)

VAGY:

GPO → User Configuration → Preferences →
Windows Settings → Folders → New Folder
Path: C:\Home\%username%
Action: Create
Másik módszer – Home mappa a felhasználói profil részeként:
ADUC → felhasználó Properties → Profile fül
Home folder: Local path → C:\Home\%username%
(Ezt érdemes felhasználónként vagy GPO Preferences-szel beállítani)
GPO hatókör szűkítése (csak Nappali képzés és Esti képzés):
Group Policy Management →
Tanulók_szabályozása GPO → Scope fül
Security Filtering:
  → Távolítsd el: Authenticated Users
  → Add: Nappali képzés
  → Add: Esti képzés
Házirendek frissítése:
powershell# PowerShell / parancssor
gpupdate /force

Képernyőkép kell: GPO részletei, a Hallgatók OU-hoz való hivatkozás, a gpupdate /force kimenet.
