---
lab:
  title: "Configurer et déployer des pipelines CI/CD pour les projets relatifs à la base de données Azure\_SQL"
  module: Develop for an Azure SQL Database
---

# Configurer et déployer des pipelines CI/CD pour les projets relatifs à la base de données Azure SQL

Dans cet exercice, vous allez créer, configurer et déployer des pipelines CI/CD pour des projets relatifs à la base de données Azure SQL à l’aide de Visual Studio Code et GitHub Actions. Ce module vous permet de vous familiariser avec le processus de configuration des pipelines CI/CD pour les projets relatifs à la base de données Azure SQL.

Cet exercice devrait prendre environ **30** minutes.

## Avant de commencer

Avant de commencer cet exercice, vous devez avoir :

- Un abonnement Azure avec les autorisations appropriées pour créer et gérer des ressources.
- [Visual Studio Code](https://code.visualstudio.com/download) installé sur votre ordinateur avec les extensions suivantes :
  - [Projets SQL Database](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql).
  - [Demandes de tirage GitHub](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github).
- Comptes GitHub.
- Connaissance de base des pipelines *GitHub Actions*.

## Créer une base de données Azure SQL

Pour commencer, vous devez créer une base de données Azure SQL.

1. Connectez-vous au [portail Azure](https://portal.azure.com?azure-portal=true). 
1. Accédez à la page **Azure SQL**, puis sélectionnez **+ Créer**.
1. Sélectionnez **SQL Database**, *Base de données unique* et le bouton **Créer**.
1. Renseignez les informations requises dans la boîte de dialogue **Créer une base de données SQL** et sélectionnez **OK**, en laissant toutes les autres options sur leurs paramètres par défaut.

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
1. Sélectionnez l’option **Autoriser les services et les ressources Azure à accéder à ce serveur**. Cette option permet à GitHub Actions d’accéder à la base de données.

    > **Remarque :** dans un environnement de production, limitez l’accès uniquement aux adresses IP nécessaires. En outre, envisagez d’utiliser des identités managées pour que votre GitHub Action puisse accéder à la base de données au lieu de l’authentification SQL. Pour plus d’informations, consultez [Identités managées dans Microsoft Entra pour Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).

1. Cliquez sur **Enregistrer**.

## Configurer un référentiel GitHub

Ensuite, vous devez configurer un nouveau dépôt GitHub.

1. Ouvrez le site web [GitHub](https://github.com).
1. Connectez-vous à votre compte GitHub.
1. Accédez à **Dépôts** sous votre compte, puis sélectionnez **Nouveau**.
1. Pour **Propriétaire**, sélectionnez votre compte. Entrez le nom **my-sql-db-repo**.
1. Définissez le dépôt sur **Privé**.
1. Cliquez sur **Create repository** (Créer le dépôt).

### Installer les extensions Visual Studio Code et cloner le dépôt

Avant de cloner le dépôt, vérifiez que vous avez installé les extensions **Visual Studio Code** nécessaires. Reportez-vous à la section **Avant de commencer** pour obtenir des conseils.

1. Dans Visual Studio Code, sélectionnez **Affichage** > **Palette de commandes**.
1. Dans la palette de commandes, tapez puis sélectionnez `Git: Clone`.
1. Entrez l’URL du dépôt que vous avez créé à l’étape précédente, puis sélectionnez **Cloner**. Votre URL doit suivre la forme suivante : *https://github.com/<your_account>/<your_repository>.git*.
1. Sélectionnez ou créez un dossier où stocker vos fichiers de dépôt.

## Créer et configurer un projet Azure SQL Database

Un projet Azure SQL Database dans Visual Studio vous permet de développer, de générer, de tester et de publier votre schéma et vos données de base de données. Dans cette section, vous allez créer un projet et le configurer pour vous connecter à la base de données Azure SQL que vous avez configurée précédemment.

1. Dans Visual Studio Code, sélectionnez **Affichage** > **Palette de commandes**.
1. Dans la palette de commandes, tapez puis sélectionnez `Database projects: New`.
    > **Remarque :** l’installation du service Outils SQL pour l’extension mssql peut prendre quelques minutes.
1. Sélectionnez **Azure SQL Database**.
1. Entrez le nom **MyDBProj**, puis appuyez sur **Entrée** pour confirmer.
1. Sélectionnez le dossier de dépôt GitHub cloné pour enregistrer le projet.
1. Sélectionnez **Oui (recommandé)** sous **Projet de style SDK**.
    > **Remarque :** notez qu’un nouveau projet est créé avec le nom **MyDBProj**.

### Créer un fichier SQL dans le projet

Avec le projet Azure SQL Database créé, nous allons ajouter un nouveau fichier SQL au projet pour créer une table.

1. Dans Visual Studio Code, sélectionnez l’icône **Projets de base de données** située dans la barre d’activité sur la gauche.
1. Cliquez avec le bouton droit sur le nom de votre projet et sélectionnez **Ajouter une table**.
1. Nommez la table **Employees** et appuyez sur **Entrée**.
1. Remplacez le script existant par le code suivant.

    ```sql
    CREATE TABLE [dbo].[Employees]
    (
        EmployeeID INT PRIMARY KEY,
        FirstName NVARCHAR(50),
        LastName NVARCHAR(50),
        Department NVARCHAR(50)
    );
    ```

1. Fermez l’éditeur. Notez que le fichier `Employees.sql` est enregistré dans le projet.

## Valider les modifications apportées au dépôt

Maintenant que le projet Azure SQL Database a été créé et que le script de table a été ajouté au projet, nous allons valider les modifications apportées au référentiel.

1. Dans Visual Studio Code, sélectionnez l’icône **Contrôle de code source** située dans la barre d’activité sur la gauche.
1. Entrez le message *Projet créé et script de création de table ajouté*.
1. Sélectionnez **Valider** pour valider les changements.
1. Sous les points de suspension, sélectionnez **Envoyer (Push** ) pour envoyer les modifications au dépôt.

## Vérifier les modifications dans le dépôt

Maintenant que vous avez envoyé les modifications, vérifions-les dans le dépôt GitHub.

1. Ouvrez le site web [GitHub](https://github.com).
1. Accédez au dépôt **my-sql-db-repo**.
1. Sous l’onglet **<> Code**, ouvrez le dossier **MyDBProj**.
1. Vérifiez si les modifications apportées au fichier **Employees.sql** sont à jour.

## Configurer l’intégration continue (CI) avec GitHub Actions

GitHub Actions vous permet d’automatiser, de personnaliser et d’exécuter vos workflows de développement logiciel directement à partir de votre dépôt GitHub. Dans cette section, vous allez configurer un workflow GitHub Actions pour générer et tester votre projet Azure SQL Database en créant une table dans la base de données.

### Créer un principal de service

1. Sélectionnez l’icône **Cloud Shell** dans le coin supérieur droit du portail Azure. Il ressemble au symbole `>_`. Si vous y êtes invité, choisissez **Bash** comme type d’interpréteur de commandes.

1. Exécutez la commande suivante dans le terminal Cloud Shell. Remplacez les valeurs `<your_subscription_id>` et `<your_resource_group_name>` par vos valeurs réelles. Vous pouvez obtenir ces valeurs sur les pages **Abonnements** et **Groupes de ressources** sur le portail Azure.

    ```azurecli
    az ad sp create-for-rbac --name "MyDBProj" --role contributor --scopes /subscriptions/<your_subscription_id>/resourceGroups/<your_resource_group_name>
    ```

    Ouvrez un éditeur de texte et utilisez la sortie de la commande précédente pour créer un extrait de code d’informations d’identification similaire à ceci :
    
    ```
    {
    "clientId": <your_service_principal_appId>,
    "clientSecret": <your_service_principal_password>,
    "tenantId": <your_service_principal_tenant>,
    "subscriptionId": <your_subscription_id>
    }
    ```

1. Laissez l’éditeur de texte ouvert. Vous en aurez besoin dans la section suivante.

### Ajouter des secrets au dépôt

1. Dans le dépôt GitHub, sélectionnez **Paramètres**.
1. Sélectionnez **Secrets et variables **, puis **Actions**.
1. Sur l’onglet **Secrets**, sélectionnez **Nouveau secret de dépôt** et entrez les informations suivantes.

    | Nom | Valeur |
    | --- | --- |
    | AZURE_CREDENTIALS | Sortie du principal de service copiée dans la section précédente.|
    | AZURE_CONN_STRING | Votre chaîne de connexion. |
   
    Votre chaîne de connexion doit ressembler à ceci :

    ```Server=<your_sqldb_server>.database.windows.net;Initial Catalog=MyDB;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;Encrypt=True;Connection Timeout=30;```

### Créer un workflow GitHub Actions

1. Dans le dépôt GitHub, sélectionnez l’onglet **Actions**.
1. Sélectionnez le lien pour **configurer vous-même un workflow**.
1. Copiez le code ci-dessous dans votre fichier **main.yml**. Le code inclut les étapes de création et de déploiement de votre projet de base de données.

    ```yaml
    name: Build and Deploy SQL Database Project
    on:
      push:
        branches:
          - main
    jobs:
      build:
        permissions:
          contents: 'read'
          id-token: 'write'
          
        runs-on: ubuntu-latest  # Can also use windows-latest depending on your environment
        steps:
          - name: Checkout repository
            uses: actions/checkout@v3

        # Install the SQLpackage tool
          - name: sqlpack install
            run: dotnet tool install -g microsoft.sqlpackage
    
          - name: Login to Azure
            uses: azure/login@v1
            with:
              creds: ${{ secrets.AZURE_CREDENTIALS }}
    
          # Build and Deploy SQL Project
          - name: Build and Deploy SQL Project
            uses: azure/sql-action@v2.3
            with:
              connection-string: ${{ secrets.AZURE_CONN_STRING }}
              path: './MyDBProj/MyDBProj.sqlproj'
              action: 'publish'
              build-arguments: '-c Release'
              arguments: '/p:DropObjectsNotInSource=true'  # Optional: Customize as needed
      ```

      L’étape **Générer et déployer le projet SQL** dans votre fichier YAML se connecte à votre base de données Azure SQL à l’aide de la chaîne de connexion stockée dans le secret `AZURE_CONN_STRING`. L’action spécifie le chemin d’accès à votre fichier projet SQL, définit l’action à publier pour déployer le projet et inclut des arguments de build à compiler en mode Mise en production. Elle utilise aussi l’argument `/p:DropObjectsNotInSource=true` pour s’assurer que les objets non présents dans la source sont supprimés de la base de données cible pendant le déploiement.

1. Validez les modifications :

### Tester le workflow GitHub Actions

1. Dans le dépôt GitHub, sélectionnez l’onglet **Actions**.
1. Sélectionnez le workflow **Générer et déployer le projet SQL Database**.
    > **Remarque :** vous verrez la progression du workflow. Patientez jusqu’à la fin de son exécution. S’il est déjà terminé, sélectionnez la dernière exécution pour afficher les détails.

### Vérifier les modifications apportées à Azure SQL Database

Maintenant que le workflow GitHub Actions est configuré pour générer et déployer votre projet Azure SQL Database, vous pouvez vérifier les modifications apportées à votre base de données Azure SQL.

1. Connectez-vous au [portail Azure](https://portal.azure.com?azure-portal=true). 
1. Accédez à la base de données SQL **MyDB**.
1. Sélectionnez **Éditeur de requêtes**.
1. Connectez-vous à la base de données à l’aide des informations d’identification **sqladmin**.
1. Dans la section **Tables**, vérifiez que la table **Employees** a été créée. Actualisez la page si nécessaire.

Vous avez réussi à configurer un workflow GitHub Actions pour générer et déployer votre projet Azure SQL Database.

## Nettoyer

Lorsque vous travaillez dans votre propre abonnement, il est recommandé, à la fin de chaque projet, de déterminer si vous avez toujours besoin des ressources que vous avez créées. 

Laisser inutilement des ressources en cours d’exécution peut entraîner des coûts supplémentaires. Vous pouvez supprimer les ressources individuellement ou supprimer le groupe de ressources dans le [Portail Azure](https://portal.azure.com?azure-portal=true).

## Plus d’informations

Si vous souhaitez en savoir plus sur l’extension SQL Database Projects pour Azure SQL Database, veuillez consulter [Prise en main de l’extension SQL Database Projects](https://learn.microsoft.com/azure-data-studio/extensions/sql-database-project-extension-getting-started?azure-portal=true).
