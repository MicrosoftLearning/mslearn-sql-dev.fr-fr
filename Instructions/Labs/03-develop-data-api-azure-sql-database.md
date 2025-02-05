---
lab:
  title: "Développer une API Données pour Azure\_SQL\_Database"
  module: Develop a Data API for Azure SQL Database
---

# Développer une API Données pour Azure SQL Database

Dans cet exercice, vous développez et déployez une API Données pour une base de données Azure SQL avec Azure Static Web Apps. Vous allez vous exercer à définir une configuration du générateur d’API de données et à la déployer dans un environnement Azure statique d’application web.

## Prérequis

Avant de commencer cet exercice, veillez à remplir les conditions préalables suivantes :

- Un abonnement Azure actif.
- Connaissance de base d’Azure SQL Database, d’Azure Static Web Apps et de GitHub.
- Visual Studio Code installé avec les extensions requises.
- Un compte GitHub pour la gestion du dépôt.

## Configurer l’environnement

Vous devez effectuer quelques étapes afin de configurer l’environnement pour cet exercice.

### Installer les extensions Visual Studio Code

Avant de commencer l’exercice, vous devez installer les extensions Visual Studio Code.

1. Ouvrez Visual Studio Code.
1. Ouvrez une fenêtre de terminal dans Visual Studio Code.
1. Installez l’interface CLI de Static Web Apps en utilisant la commande suivante :

    ```bash
    npm install -g @azure/static-web-apps-cli
    ```

1. Installez l’interface CLI de Générateur d’API de données en utilisant la commande suivante :

    ```bash
    dotnet tool install --global Microsoft.DataApiBuilder
    ```

Visual Studio Code est maintenant configuré avec les extensions nécessaires.

### Créer une base de données Azure SQL

Si ce n’est déjà fait, vous devez créer une base de données Azure SQL.

1. Connectez-vous au [portail Azure](https://portal.azure.com?azure-portal=true). 
1. Accédez à la page **Azure SQL**, puis sélectionnez **+ Créer**.
1. Sélectionnez **SQL Database**, *Base de données unique* et le bouton **Créer**.
1. Renseignez les informations requises dans la boîte de dialogue **Créer une base de données SQL** et sélectionnez **OK** (laissez toutes les autres options sur leurs valeurs par défaut).

    | Paramètre | Valeur |
    | --- | --- |
    | Offre serverless gratuite | *Appliquer l’offre* |
    | Abonnement | Votre abonnement |
    | Resource group | *Sélectionner ou créer un groupe de ressources* |
    | Nom de la base de données | *MyDB* |
    | Serveur | *Sélectionner le lien **Créer nouveau*** |
    | Nom du serveur | *Choisissez un nom unique* |
    | Emplacement | *Sélectionner un emplacement* |
    | Méthode d’authentification | *Utilisez l’authentification SQL* |
    | Connexion d’administrateur du serveur | *sqladmin* |
    | Mot de passe | *Entrez un mot de passe* |
    | Confirmer le mot de passe | *Confirmez le mot de passe* |

1. Sélectionnez **Vérifier + créer**, puis **Créer**.
1. Une fois le déploiement terminé, accédez au *serveur* Azure SQL Database que vous avez créé.
1. Dans le volet de navigation gauche, sous **Sécurité**, sélectionnez **Mise en réseau**. Ajoutez votre adresse IP aux règles de pare-feu.
1. Cliquez sur **Enregistrer**.

### Ajouter des exemples de données à la base de données

Maintenant que vous disposez d’une base de données Azure SQL, vous devez ajouter des exemples de données. Cela vous aidera à tester l’API une fois qu’elle est opérationnelle.

1. Accédez à votre nouvelle base de données Azure SQL Database.
1. Utilisez l’**éditeur de requête** du portail Azure pour exécuter le script SQL suivant :

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    
    INSERT INTO [dbo].[Employees] VALUES (1,'John', 'Smith', 'Accounting');
    INSERT INTO [dbo].[Employees] VALUES (2,'Michelle', 'Sanchez', 'IT');
    
    SELECT * FROM [dbo].[Employees];
    ```

### Créer une application web de base dans GitHub

Avant de pouvoir créer une application web statique Azure, nous devons créer une application web de base dans GitHub.

1. Pour créer une application web de base dans GitHub, accédez au site web [Générer un dépôt standard](https://github.com/staticwebdev/vanilla-basic/generate).
1. Vérifiez que le modèle de dépôt est défini sur **staticwebdev/vanilla-basic**.
1. Pour ***Propriétaire***, sélectionnez votre compte GitHub.
1. Sous ***Nom du dépôt***, entrez le nom **my-sql-repo**.
1. Définissez le dépôt sur **Privé**.
1. Sélectionnez le bouton **Créer un dépôt**.

## Créer une application web statique Azure

Nous allons d’abord créer notre application web statique, puis y ajouter la configuration du Générateur d’API de données.

1. Dans le portail Azure, accédez à la page **Static Web Apps**.
1. Sélectionnez **+ Créer**.
1. Renseignez les informations suivantes dans la boîte de dialogue **Créer une application web statique** (laissez toutes les autres options sur leurs valeurs par défaut) :

    | Paramètre | Valeur |
    | --- | --- |
    | Abonnement | Votre abonnement |
    | Resource group | *Sélectionner ou créer un groupe de ressources* |
    | Nom | *Un nom unique* |
    | Source du plan d’hébergement | *GitHub* |
    | Compte GitHub | *Sélectionner votre compte* |
    | Organization | *Probablement votre nom d’utilisateur GitHub* |
    | Dépôt | *Sélectionner le dépôt que vous avez créé à l’étape précédente* |
    | Branche | *main* |

1. Sélectionnez **Vérifier + créer**, puis **Créer**.
1. Une fois le déploiement terminé, accédez à la ressource.
1. Sélectionnez le bouton **Afficher l’application dans le navigateur**. Vous devez voir une page web simple avec un message de bienvenue. Vous pouvez fermer cet onglet.

## Ajouter le fichier de configuration du générateur d’API de données

Il est temps d’ajouter la configuration du générateur d’API de données à l’application web statique Azure. Nous devons créer un fichier dans le dépôt GitHub pour ajouter la configuration du générateur d’API de données.

1. Dans Visual Studio Code, clonez le dépôt GitHub que vous avez créé précédemment.
1. Ouvrir une fenêtre de terminal dans Visual Studio Code.
1. Exécutez la commande suivante pour créer un fichier de configuration du générateur d’API de données :

    ```bash
    swa db init --database-type "mssql"
    ```

    Cela crée un dossier nommé *swa-db-connections* et un fichier nommé *staticwebapp.database.config.json* à l’intérieur de ce dossier.

1. Exécutez la commande suivante pour ajouter les entités de base de données au fichier de configuration :

    ```bash
    dab add "Employees" --source "dbo.Employees" --permissions "anonymous:*" --config "swa-db-connections/staticwebapp.database.config.json"
    ```

1. Examinez le contenu du fichier *staticwebapp.database.config.json*. 
1. Validez et envoyez (push) les modifications dans le référentiel Git.

## Configurer la connexion à la base de données

1. Dans le portail Azure, accédez à l’application web statique Azure que vous avez créée.
1. Dans la section **Paramètres**, sélectionnez **Connexion de base de données**.
1. Sélectionnez **Lier une base de données existante**.
1. Dans la boîte de dialogue **Lier la base de données*, sélectionnez la base de données Azure SQL que vous avez créée précédemment avec les paramètres supplémentaires suivants.

    | Paramètre | Valeur |
    | --- | --- |
    | Type de base de données | *Azure SQL Database* |
    | Type d’authentification | *Chaîne de connexion* |
    | Nom d’utilisateur | *Nom de votre utilisateur administrateur* |
    | Mot de passe | *Mot de passe que vous avez donné à votre utilisateur administrateur* |
    | Case à cocher Accusé de réception | *Activée* |

   > **Remarque :** dans un environnement de production, limitez l’accès uniquement aux adresses IP nécessaires. En outre, envisagez d’utiliser des identités managées pour que votre application web statique puisse accéder à la base de données au lieu de l’authentification SQL. Pour plus d’informations, consultez [Identités managées dans Microsoft Entra pour Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).
1. Sélectionnez **Lien**.

## Tester le point de terminaison de l’API Données

À présent, nous devons uniquement tester le point de terminaison de l’API Données.

1. Dans le portail Azure, accédez à l’application web statique Azure que vous avez créée.
1. Dans la page Vue d’ensemble, copiez l’URL de l’application web.
1. Ouvrez un nouvel onglet de navigateur, puis collez l’URL. Vous devez toujours voir la page web simple avec le message **Vanille JavaScript App**.
1. Ajoutez **/data-api** à la fin de l’URL, puis appuyez sur **Entrée**. La mention **Healthy** (Sain) doit s’afficher pour indiquer que l’API Données fonctionne.
1. Ajoutez **/data-api/rest/Employees** à la fin de l’URL, puis appuyez sur **Entrée**. Vous devez voir les exemples de données que vous avez ajoutés à Azure SQL Database précédemment.

Vous avez réussi à développer et à déployer une API Données pour une base de données Azure SQL avec Azure Static Web Apps.

## Nettoyer

Lorsque vous travaillez dans votre propre abonnement, il est recommandé, à la fin de chaque projet, de déterminer si vous avez toujours besoin des ressources que vous avez créées. 

Laisser inutilement des ressources en cours d’exécution peut entraîner des coûts supplémentaires. Vous pouvez supprimer les ressources individuellement ou supprimer le groupe de ressources dans le [Portail Azure](https://portal.azure.com?azure-portal=true).

## Plus d’informations

Pour plus d’informations sur le générateur d’API de données pour les bases de données Azure SQL Database, consultez [Qu’est-ce que le générateur d’API de données pour les bases de données Azure ?](https://learn.microsoft.com/azure/data-api-builder/overview?azure-portal=true).
