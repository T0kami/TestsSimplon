# Brief partie 3 - Configuration Windows server du parc Rue25

**Contexte** : L’agence Rue25, spécialisée dans l’immobilier de luxe à l’île de La Réunion, ouvre une nouvelle activité : la gestion d’hôtels et de gîtes de prestige.  
Cette documentation permettra de réaliser pas à pas le serveur Windows de l'agence, contenant un service DHCP et ADDS sous VirtualBox, ainsi que les UO, groupes et utilisateurs.

Nous resterons dans une démarche simple pour l'exercice, les procédures se feront via l'interface graphique principalement.

## Diagramme réseau Rue25

![enter image description here](https://i.postimg.cc/c6yKMzS2/planip.png)

Toujours dans une optique de simplicité pour l'exercice, le réseau restera simple (pas de VLAN, un switch...).

----------
# 1. Préparation de l’environnement

Téléchargez l'ISO de Windows Server 2022 à cette adresse: 
https://info.microsoft.com/ww-landing-windows-server-2022.html

- Installez et exécutez VirtualBox
- Cliquez sur le bouton bleu "Nouvelle" pour créer une machine. Sélectionnez les options comme l'image ci-dessous :
![enter image description here](https://i.postimg.cc/fb2fjMgP/creation-machine-1-def.png)
- Choisissez un nom d'utilisateur/mot de passe sécurisé
- Allouez des ressources suffisantes selon votre configuration
- Stockage : environ 40go en disque VDI dynamique

Créez et lancez la machine, l'installation de Windows devrait s'effectuer automatiquement.
Une fois terminée, vous pouvez créer un snapshot sur VirtualBox pour disposer d'un point de restauration propre.

# 2. Configuration du Serveur Windows

**Attribution d'une IP statique au serveur**
Nous allons commencer par attribuer une adresse IP à notre serveur.
- Cela peut se faire simplement en ligne de commande via l'utilitaire "sconfig" > option n°8.

- Ou bien manuellement : 
	1. Démarrer > Settings > Network & Internet > Advanced Network settings
	2. Configurer la carte
	   - Clic droit sur Ethernet > Properties
	   - Double-cliquez  sur Internet Protocol version 4 (TCP/IPv4) 
	   - Cochez "Use the following IP" et saisissez les valeurs ci-dessous.  
	   - Validez 
	   
	   
| Paramètre | Valeur conseillée |
|-----------|------------------|
| **Adresse IP** | `192.168.10.2` |
| **Masque** | `255.255.255.0` |
| **Passerelle** | `192.168.10.1` |
| **DNS préféré** | `192.168.10.2` (le serveur lui-même, pourra servir de DNS plus tard) |

# 3. Installation et configuration du rôle DHCP

## 3.1 Ajout du rôle DHCP via le Gestionnaire de serveur

1. Ouvrir _Server Manager  
   Démarrer > Server Manager  

2. Lancer l’Assistant 
   Menu **Manage** > **Add Roles and Features**.  
   - _Installation type_ : **Role-based or feature-based installation**  
   - _Destination server_ : laisser votre serveur sélectionné.

3. Sélectionner le rôle 
   - Cochez **DHCP Server**.  
   - Lorsque la boîte de dialogue s’affiche, cliquez sur **Add Features** puis **Next**

4. Parcourir les pages _Features_ et _DHCP Server_  
   Laissez les choix par défaut, cliquez sur **Next** jusqu’à **Install**.

5. Fermez l’assistant avec **Close** une fois l’opération terminée.

6. Configuration du DHCP 
   - Dans la barre d’état du _Server Manager_, cliquez sur **Complete DHCP configuration**.  
   - **Autorisation** : comme le serveur n’est pas encore membre du domaine `rue25.com`, **désactivez** la case _Authorize this DHCP server in AD DS_ pour le moment ; vous l’autoriserez après la promotion en contrôleur de domaine.  
   - Cliquez sur **Commit** puis **Close**.

## 3.2 Création et configuration de la plage d’adresses (Scope)

1. Ouvrir la console DHCP  
   Démarrer > **Windows Administrative Tools** > **DHCP**.

2. Créer l’étendue IPv4 
   - Développez le serveur > clic droit **IPv4** > **New Scope**
   - Name : LAN_Rue25 - Description : Plage bureau & portables  
   - **IP Address Range** :  
     - _Start IP_ : 192.168.10.100  
     - _End IP_ : 192.168.10.200  
     - _Subnet mask_ : 255.255.255.0 
   - **Add Exclusions and Delay** : facultatif 
   - **Lease Duration** : 8 days (valeur par défaut adaptée).  
   - Choisissez **Yes, I want to configure these options now**. 

3. Activer l’étendue  
   À l’étape _Activate Scope_, sélectionnez **Yes, I want to activate this scope** puis **Finish**.

## 3.3 Options DHCP

Dans l’Assistant qui se poursuit automatiquement :

| Page de l’assistant | Valeur à saisir | Commentaire |
|---------------------|-----------------|-------------|
| **Router (Option 003)** | `192.168.10.1` | IP de la box ADSL|
| **Domain Name and DNS Servers (Options 006 & 015)** | - _Parent domain_ : rue25.com  <br> - _IP address #1_ : 192.168.10.2 | Les clients utiliseront directement le futur contrôleur AD comme DNS |
| **WINS Servers** | Laisser vide | Non utilisé dans ce projet |

Cliquez sur **Next** à chaque page, puis **Finish** pour quitter l’assistant.

# 4. Installation et configuration du rôle **AD DS**
1. Pour ajouter le rôle, faites comme pour l'installation de DHCP, mais cochez "Active Directory Domain Services".
![enter image description here](https://i.postimg.cc/Bt9gLNwQ/add-roles.png)
- Procédez à l'installation.

2. Après l'installation  
   - Cliquez sur le triangle jaune du **Server Manager**, puis sur
     **Promote this server to a domain controller**.  
     ![enter image description here](https://i.postimg.cc/tT8fk3m8/ADDS-config.png)

- L’assistant de Configuration AD DS s’ouvre, configurez comme suit :

| Page de l’assistant | Paramètres à saisir |
|---------------------|---------------------|
| **Deployment Configuration** | • Add a new forest, domain name_ : **rue25.com** |
| **Domain Controller Options** | • _Forest & Domain functional level_ : Windows Server 2022 (par défaut)<br>• Cochez Domain Name System (DNS) server et Global Catalog<br>• Saisissez un mot de passe DSRM complexe|
| **DNS Options** | Ignorez l’avertissement  |
| **Additional Options** | Laissez le nom NetBIOS proposé (RUE25) |
| **Paths** | Conservez les emplacements par défaut :<br>`C:\Windows\NTDS` & `C:\Windows\SYSVOL` |
| **Review / Prerequisites Check** | Cliquez sur Install. |

Le serveur redémarre automatiquement à la fin.  

3. Connectez-vous ensuite avec le compte **Administrator** (domaine _rue25_). 
4. Authorisez le role DHCP dans AD : 
	- **DHCP Manager** > **Clic droit sur DHCP** > **Manage authorized servers**

# 5.  Création des Unités organisationnelles (UO)

- **Démarrer > Windows Administrative Tools > Active Directory Users and Computers**  
- Dans **View**, cochez **Advanced Features** pour afficher tous les conteneurs système.

1. Clic droit sur le domaine `rue25.com` > New > Organizational Unit. 
2. Créez toutes les UO nécessaires.

**Arborescence recommandée**

rue25.com
└───RUE25
├─── _Groups
├───  _Users
	- Direction
	- Consultants
	- Commerciaux
	- Comptables
	- Secretariat

> *Pourquoi une racine « RUE25 » ?*   
> Séparer les objets créés de ceux par défaut simplifie les sauvegardes, les GPO et l’audit.

# 6.  Création des groupes de sécurité
1. Ouvrez l’UO **_Groups**.  
2. **Clic droit > New > Group**.  
3. Renseignez :
   - **Group name** : `G_Direction` (par exemple)  
   - **Group scope** : *Global*  
   - **Group type** : *Security*  
4. Validez par "OK"
5. Répétez pour : `G_Consultants`, `G_Commerciaux`, `G_Comptables`, `G_Secretariat`.

>Les groupes **Global** contiennent des comptes du même domaine et sont utilisables dans les ACL sur tout le domaine.

# 7. Création des comptes utilisateurs

1. Dans l’UO correspondant au service (**UO_Direction** pour Samira Bien) :  
   **clic droit > New > User**.  
2. Renseignez **First name**, **Last name**, **User logon name** > **Next**.  
3. Saisissez un **mot de passe temporaire** et cochez **User must change password at next logon** > **Next > Finish**.  
4. **Ajouter l’utilisateur à son groupe** :  
   - Double-cliquez le compte > onglet **Member Of** > **Add** > tapez `G_Direction` > **Check Names > OK**.

Recommencez pour chaque utilisateur et son groupe.

# 8. Partage de fichiers et gestion des droits

1. **Créer un volume dédié**  
   - Dans **Server Manager > File and Storage Services > Volumes**, créez (ou montez) le disque **D:** destiné aux données  (séparer données et système facilite les sauvegardes)  
Pour ce tutoriel cependant, nous allons rester sur le disque C:

2. Créez un dossier "Public" puis des sous-dossiers pour chaque service (direction, comptables...)

**Autorisations NTFS**

 Procédure détaillée pour un dossier (exemple Direction)

1. **Clic droit** sur `D:\Public\Direction` > **Propriétés** > onglet
**Sécurité**.  
2. **Avancé** > **Désactiver l’héritage** > **Convertir**.  
3. **Supprimer** l’entrée **Everyone** si elle existe.  
4. **Ajouter** > **Sélectionner un principal** : tapez `G_Direction` > **Vérifier les noms** > **OK**.  
5. Dans **Autorisations de base**, cochez **Modifier**.  
6. Vérifiez que **Domain Admins** possède **Contrôle total**.  
7. **Appliquer** puis **OK**.

Répétez la même séquence pour chaque dossier en remplaçant le groupe

**Création du partage**

1. Server Manager > File and Storage Services > Shares  
2. TASKS > New Share > **SMB Share – Quick** :  
- Share location : `D:\Public`  
- Share name : `PUBLIC`
3. Cochez Enable access-based enumeration (cache la vue des dossiers non autorisés)
4. Permissions (assistant) :  
-  Supprimez l’entrée **Everyone**.  
- **Add** `G_Direction`, `G_Consultants`, … : cochez **Change**.  
- **Add** `Domain Admins` : cochez **Full Control**.  
5. "Create" pour finaliser le partage.

# Conclusion

Nous disposons désormais d’un serveur avec les rôles, groupes et configurations essentielles. Cette base couvre les besoins immédiats et reste prête à être étendue et sécurisée. 

Espérant que vous apprécierez lire autant que je me suis amusé à le faire!
