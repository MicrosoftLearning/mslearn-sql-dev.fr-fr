---
lab:
  title: Importer et exporter des données pour le développement dans Azure SQL Database
  module: Import and export data for development in Azure SQL Database
---

# Importer et exporter des données pour le développement dans Azure SQL Database

Dans cet exercice, vous importez des données à partir d’un point de terminaison REST externe (simulé à l’aide d’Azure Static Web App) et exportez des données à l’aide d’une fonction Azure. Le labo fournit une expérience pratique de l’utilisation d’Azure SQL Database à des fins de développement, en mettant l’accent sur l’intégration d’API REST et d’Azure Functions pour gérer les opérations d’importation/exportation de données.

## Prérequis

Avant de commencer ce labo, veillez à remplir les conditions préalables suivantes :

- Un abonnement Azure actif avec des autorisations pour créer et gérer des ressources
- Connaissances de base d’Azure SQL Database, des API REST et d’Azure Functions
- Visual Studio Code installé avec les extensions suivantes :
      - Extension Azure Functions
- Git installé pour cloner le dépôt
- SQL Server Management Studio (SSMS) ou Azure Data Studio pour gérer la base de données

## Configurer l’environnement

Commençons par configurer les ressources nécessaires pour ce labo, notamment une base de données Azure SQL et les outils nécessaires pour importer et exporter des données.

### Créer une base de données Azure SQL

Dans cette étape, vous allez créer une base de données dans Azure :

1. Dans le portail Azure, accédez à la page **Bases de données SQL**.
1. Sélectionnez **Créer**.
1. Renseignez les champs obligatoires :

    | Paramètre | Valeur |
    |---|---|
    | Offre serverless gratuite | Appliquer l’offre |
    | Abonnement | Votre abonnement |
    | Resource group | Sélectionner ou créer un groupe de ressources |
    | Nom de la base de données | **MyDB** |
    | Serveur | Sélectionner ou créer un serveur |
    | Méthode d’authentification | Authentification SQL |
    | Connexion d’administrateur serveur | **sqladmin** |
    | Mot de passe | Entrer un mot de passe sécurisé |
    | Confirmer le mot de passe | Confirmer le mot de passe |

1. Sélectionnez **Vérifier + créer**, puis **Créer**.
1. Une fois le déploiement terminé, accédez à la section **Mise en réseau** de votre ***serveur Azure SQL*** (et non la base de données Azure SQL Database) et :
    1. Ajoutez votre adresse IP aux règles de pare-feu. Cela vous permettra d’utiliser SQL Server Management Studio (SSMS) ou Azure Data Studio pour gérer la base de données.
    1. Cochez la case **Autoriser les services et les ressources Azure à accéder à ce serveur**. Cela permettra à l’application Azure Function d’accéder au serveur de base de données.
    1. Enregistrez les changements apportés.
1. Accédez à la section **Microsoft Entra ID** de votre **serveur Azure SQL** et veillez à *désélectionner* **Prendre en charge l’authentification Microsoft Entra uniquement pour ce serveur** et **enregistrer** vos modifications si cette option est sélectionnée. Cet exemple utilise l’authentification SQL. Nous devons donc désactiver la prise en charge d’Entra uniquement.

> [!NOTE]
> Dans un environnement de production, vous devez déterminer le type d’accès et à partir d’où vous souhaitez accorder l’accès. Bien que la fonction ait une légère modification si vous choisissez l’authentification Entra uniquement, notez que vous devrez activer *Autoriser les services et les ressources Azure à accéder à ce serveur* pour permettre à l’application Azure Function d’accéder au serveur.

### Cloner le dépôt GitHub

1. Ouvrez **Visual Studio Code**.

1. Clonez le dépôt GitHub et préparez votre projet :

    1. Dans **Visual Studio Code**, ouvrez la **Palette de commandes** en sélectionnant **Ctrl+Maj+P** (Windows) ou **Cmd+Maj+P** (Mac).
    1. Tapez **Git: Clone** et sélectionnez **Git: Clone**.
    1. Dans l’invite, entrez l’URL suivante pour cloner le dépôt :
        ```bash
        https://github.com/MicrosoftLearning/mslearn-sql-dev.git
        ```

    1. Choisissez le dossier de destination dans lequel vous souhaitez cloner le dépôt.

### Configurer le stockage Blob Azure pour les données JSON

Nous allons maintenant configurer **Stockage Blob Azure** pour héberger le fichier **employees.json**. Suivez ces étapes dans le portail Azure et **Visual Studio Code**.

Commençons par créer un compte de stockage Azure.

1. Dans le **Portail Azure**, accédez à la page **Comptes de stockage**.
1. Sélectionnez **Créer**.
1. Renseignez les champs obligatoires :

    | Paramètre | Valeur |
    |---|---|
    | Abonnement | Votre abonnement |
    | Resource group | Sélectionner ou créer un groupe de ressources |
    | Nom du compte de stockage | Choisir un nom unique au monde |
    | Région | Choisir la région la plus proche de vous |
    | Service principal | **Stockage Blob Azure ou Azure Data Lake Storage Gen2** |
    | Performances | standard |
    | Redondance | Stockage localement redondant (LRS) |

1. Sélectionnez **Vérifier + créer**, puis **Créer**.
1. Attendez que le compte de stockage soit créé.

Maintenant que nous avons un compte, nous allons charger **employees.json** dans le stockage Blob.

1. Dans le portail Azure, accédez à la page **Comptes de stockage**.
1. Sélectionnez votre compte de stockage.
1. Accédez à la section **Conteneurs**.
1. Créez un conteneur nommé **jsonfiles**.
1. Dans le conteneur, cliquez sur **Charger** et chargez le fichier **employees.json** situé sous **/Allfiles/Labs/04/blob-storage** dans le répertoire cloné.

Bien qu’il soit possible d’autoriser l’accès anonyme au fichier, dans notre cas, nous allons générer une *signature d’accès partagé (SAP)* pour ce fichier afin de garantir un accès sécurisé.

1. Dans le conteneur **jsonfiles**, sélectionnez le fichier **employees.json**.
1. Sélectionnez **Générer une signature d’accès partagé** dans le menu contextuel.
1. Passez en revue les paramètres, puis sélectionnez **Générer une SAP et une URL**.
1. Un jeton SAP de blob et une URL SAP de blob sont générés. Copiez les valeurs des champs **Jeton SAP de blob** et **URL SAP de blob**, car vous en aurez besoin au cours des étapes suivantes. Vous ne pourrez plus accéder à la valeur du jeton une fois que vous aurez fermé cette fenêtre.

Nous devrions maintenant disposer d’une URL sécurisée pour accéder au fichier **employees.json**. Testons-la.

1. Ouvrez un nouvel onglet de navigateur, puis collez l’**URL SAP de blob**.
1. Vous devez voir le contenu du fichier **employees.json** affiché dans le navigateur, qui doit ressembler à ceci :

    ```json
    {
        "employees": [
            {
                "EmployeeID": 1,
                "FirstName": "John",
                "LastName": "Doe",
                "Department": "HR"
            },
            {
                "EmployeeID": 2,
                "FirstName": "Jane",
                "LastName": "Smith",
                "Department": "Engineering"
            }
        ]
    }
    ```

### Importer des données de stockage Blob vers Azure SQL Database

Nous sommes maintenant prêts à importer les données à partir du fichier **employees.json** hébergé sur le stockage Blob Azure dans notre base de données Azure SQL.

Nous devons commencer par créer une **clé principale** et des **informations d’identification délimitées à la base de données** dans la base de données Azure SQL.

1. Connectez-vous à votre base de données Azure SQL, à l’aide de **SQL Server Management Studio** (SSMS) ou **Azure Data Studio**.
1. Exécutez la commande SQL suivante pour créer une clé principale *si vous n’en avez pas déjà* :

    ```sql
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'YourStrongPassword!';
    ```

1. Ensuite, créez des **informations d’identification délimitées à la base de données** pour accéder au stockage Blob Azure en exécutant la commande SQL suivante :

    ```sql
    CREATE DATABASE SCOPED CREDENTIAL MyBlobCredential
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE', 
    SECRET = '<your-sas-token>';
    ```

    Remplacez ***<your-sas-token>*** par le **jeton SAP de blob** généré précédemment.

1. Enfin, vous avez besoin d’une **source de données** pour accéder au stockage Blob Azure. Exécutez la commande SQL suivante pour créer une **source de données** :

    ```sql
    CREATE EXTERNAL DATA SOURCE MyBlobDataSource
    WITH (
        TYPE = BLOB_STORAGE,
        LOCATION = 'https://<your-storage-account-name>.blob.core.windows.net',
        CREDENTIAL = MyBlobCredential
    );
    ```

    Remplacez ***storageAccountName*** par le nom de votre compte de stockage Azure.

Tout est maintenant configuré pour importer les données à partir du fichier **employees.json** dans la *base de données Azure SQL*.

Utilisez la commande SQL suivante pour importer des données à partir du fichier **employees.json** hébergé sur le *stockage Blob Azure* :

```sql
SELECT EmployeeID
    , FirstName
    , LastName
    , Department
INTO dbo.employee_data
FROM OPENROWSET(
    BULK 'jsonfiles/employees.json',
    DATA_SOURCE = 'MyBlobDataSource',
    SINGLE_CLOB
) AS JSONData
CROSS APPLY OPENJSON(JSONData.BulkColumn, '$.employees')
WITH (
    EmployeeID INT '$.EmployeeID',
    FirstName NVARCHAR(50) '$.FirstName',
    LastName NVARCHAR(50) '$.LastName',
    Department NVARCHAR(50) '$.Department'
) AS EmployeeData;
```

Cette commande lit le fichier **employees.json** à partir du conteneur **jsonfiles** dans le *stockage Blob Azure* et importe les données dans la table **employee_data** dans la base de données Azure SQL.

Vous pouvez maintenant exécuter la commande SQL suivante pour vérifier l'importation des données : 

```sql
SELECT * FROM dbo.employee_data;
```

Vous devez voir les données du fichier **employees.json** importé dans la table **employee_data**.

---

## Exporter des données à l’aide d’une application Azure Function

Dans cette partie du labo, vous allez créer une application Azure Function en C# pour exporter des données à partir de votre base de données Azure SQL. Cette fonction récupère les données et les retourne en tant que réponse JSON.

### Créer une application Azure Function dans Visual Studio Code

Commençons par créer une application Azure Function dans Visual Studio Code :

1. Ouvrez **Visual Studio Code**.
1. Dans le volet Explorateur, accédez au dossier **/Allfiles/Labs/04/azure-functions**.
1. Cliquez avec le bouton droit sur le nom du dossier **azure-functions**, puis sélectionnez **Ouvrir dans le terminal intégré**.
1. Dans le terminal VS Code, connectez-vous à Azure à l’aide de la commande suivante :

    ```bash
    az login
    ```

1. (Facultatif) Si vous avez plusieurs abonnements, sélectionnez l’abonnement actif :

    ```bash
    az account set --subscription <your-subscription-id>
    ```

1. Exécutez la commande suivante pour créer une application Azure Function :

    ```bash
    $functionappname = "YourUniqueFunctionAppName"
    $resourcegroup = "YourResourceGroupName"
    $location = "YourLocation"
    # NOTE - The following should be a new storage account name where your Azure function will resided.
    # It should not be the same Storage Account name used to store the JSON file
    $storageaccount = "YourStorageAccountName"

    az storage account create --name $storageaccount --location $location --resource-group $resourcegroup --sku Standard_LRS
    
    az functionapp create --resource-group $resourcegroup --consumption-plan-location $location --runtime dotnet --name  $functionappname --os-type Linux --storage-account $storageaccount --functions-version 4
    
    ```

    ***Remplacez les espaces réservés par vos propres valeurs. N’utilisez pas le nom du compte de stockage utilisé pour le fichier json, ce script doit créer un compte de stockage pour stocker l’application Azure Function***.


### Créer une application de fonction dans Visual Studio Code

Créons une fonction dans Visual Studio Code pour exporter des données à partir de la base de données Azure SQL :

Vous devrez peut-être ajouter l’extension Azure Functions à Visual Studio Code si ce n’est déjà fait. Pour ce faire, recherchez **Azure Functions** dans le volet des extensions et installez-la.

1. Dans Visual Studio Code, appuyez sur **Ctrl+Maj+P** (Windows) ou **Cmd+Maj+P** (Mac) pour ouvrir la palette de commandes.
1. Tapez et sélectionnez **Azure Functions : Créer un projet**.
1. Choisissez votre répertoire **Function App**. Choisissez le dossier **/Allfiles/Labs/04/azure-functions** du dépôt clone GitHub.
1. Choisissez **C#** comme langage.
1. Choisissez **.Net 8.0 LTS** comme runtime.
1. Sélectionnez le modèle **Déclencheur HTTP**.
1. Appelez la fonction **ExportDataFunction**.
1. Créez l’espace de noms **Contoso.ExportFunction**.
1. Donnez à la fonction le niveau d’accès **anonyme**.

### Écrire le code C# pour l’exportation des données

1. L’application Azure Function peut nécessiter l’installation préalable de quelques packages. Vous pouvez les installer en exécutant les commandes suivantes :

    ```bash
    dotnet add package Microsoft.Data.SqlClient
    dotnet add package Newtonsoft.Json
    dotnet restore
    
    npm install -g azure-functions-core-tools@4 --unsafe-perm true

    ```

1. Remplacez le code de fonction de l’espace réservé par le code C# suivant pour interroger votre base de données Azure SQL et renvoyer les résultats au format JSON :

    ```csharp
    using System;
    using System.IO;
    using Microsoft.AspNetCore.Mvc;
    using Microsoft.Azure.WebJobs;
    using Microsoft.Azure.WebJobs.Extensions.Http;
    using Microsoft.AspNetCore.Http;
    using Microsoft.Extensions.Logging;
    using Microsoft.Data.SqlClient;
    using Newtonsoft.Json;
    using System.Collections.Generic;
    using System.Threading.Tasks;
    
    public static class ExportDataFunction
    {
        [FunctionName("ExportDataFunction")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", Route = null)] HttpRequest req,
            ILogger log)
        {
            // Connection string to the database
            // NOTE: REPLACE THIS CONNECTION STRING WITH THE CONNECTION STRING OF YOUR AZURE SQL DATABASE
            string connectionString = "Server=tcp:yourserver.database.windows.net;Database=DataLabDatabase;User ID=youruserid;Password=yourpassword;Encrypt=True;";
            
            // List to hold employee data
            List<Employee> employees = new List<Employee>();
            
            try
            {
                // Establishing connection to the database
                using (SqlConnection conn = new SqlConnection(connectionString))
                {
                    await conn.OpenAsync();
                    var query = "SELECT EmployeeID, FirstName, LastName, Department FROM employee_data";
                    
                    // Executing the query
                    using (SqlCommand cmd = new SqlCommand(query, conn))
                    {
                        // Adding parameters to the query (if needed)
                        // cmd.Parameters.AddWithValue("@ParameterName", parameterValue);

                        using (SqlDataReader reader = await cmd.ExecuteReaderAsync())
                        {
                            // Reading data from the database
                            while (await reader.ReadAsync())
                            {
                                employees.Add(new Employee
                                {
                                    EmployeeID = (int)reader["EmployeeID"],
                                    FirstName = reader["FirstName"].ToString(),
                                    LastName = reader["LastName"].ToString(),
                                    Department = reader["Department"].ToString()
                                });
                            }
                        }
                    }
                }
            }
            catch (SqlException ex)
            {
                // Logging SQL errors
                log.LogError($"SQL Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
            catch (System.Exception ex)
            {
                // Logging unexpected errors
                log.LogError($"Unexpected Error: {ex.Message}");
                return new StatusCodeResult(500);
            }
    
            // Returning the list of employees as a JSON response
            return new OkObjectResult(JsonConvert.SerializeObject(employees, Formatting.Indented));
        }
    
        // Employee class to hold employee data
        public class Employee
        {
            public int EmployeeID { get; set; }
            public string FirstName { get; set; }
            public string LastName { get; set; }
            public string Department { get; set; }
        }
    }
    ```

    *N’oubliez pas de remplacer **connectionString** par la chaîne de connexion à votre base de données Azure SQL et d’entrer votre mot de passe sqladmin dans le chaîne de connexion.*

    > **Remarque :** dans un environnement de production, limitez l’accès uniquement aux adresses IP nécessaires. Envisagez aussi d’utiliser des identités managées pour que votre application Azure Function puisse accéder à la base de données au lieu de l’authentification SQL. Pour plus d’informations, consultez [Identités managées dans Microsoft Entra pour Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).

1. Enregistrez le code de fonction et vérifiez que votre fichier **.csproj** inclut le package **Newtonsoft.Json** pour sérialiser des objets au format JSON. S’il n’est pas inclus, ajoutez-le.

    ```xml
    <PackageReference Include="Newtonsoft.Json" Version="13.X.X" />
    ```

Déployons maintenant l’application Azure Function sur Azure.

### Déployer l’application Azure Function dans Azure

1. Dans le terminal intégré **Visual Studio Code**, exécutez la commande suivante pour déployer l’application Azure Functions dans Azure :

    ```bash
    func azure functionapp publish <your-function-app-name>
    ```

    Remplacez ***<your-function-app-name>*** par le nom de votre application Azure Function.

1. Attendez la fin du déploiement.

### Obtenir l’URL de l’application Azure Function

1. Ouvrez le portail Azure et accédez à votre application Azure Function.
1. Dans la section *Vue d’ensemble*, sous l’onglet *Fonction*, vous pouvez voir votre nouvelle fonction, sélectionnez-la.
1. Sous l’onglet **Code + Test**, sélectionnez **Obtenir l’URL de la fonction**.
1. Copiez la **valeur par défaut (clé de fonction)**, nous en aurons besoin un peu plus loin. L’URL doit ressembler à ce qui suit :
   
   ```url
   https://YourFunctionAppName.azurewebsites.net/api/ExportDataFunction?code=2pjO0HqRyz_13DHQg8ga-ysdDWbDU_eHdtlixbAHLVEGAzFuplomUg%3D%3D
   ```

### Tester l’application Azure Function

1. Une fois le déploiement terminé, vous pouvez tester la fonction en envoyant une requête HTTP à l’URL de clé de la fonction que vous avez copiée précédemment à partir du terminal Visual Studio Code :

    ```bash
    curl https://<your-function-app-name>.azurewebsites.net/api/ExportDataFunction?code=<the function key embedded to your function URL>
    ```

1. La réponse doit contenir les données exportées de votre table ***employee_data*** au format JSON.

Bien que cette fonction soit un exemple simple, vous pouvez l’étendre pour inclure une logique et un traitement de données plus complexes, tels que le filtrage, le tri et l’agrégation de données, etc.  Votre code peut également être étendu pour inclure la gestion des erreurs, la journalisation et des fonctionnalités de sécurité.

### Nettoyer les ressources

Une fois le labo terminé, vous pouvez supprimer les ressources créées dans cet exercice pour éviter des coûts supplémentaires :

- Supprimez la base de données Azure SQL.
- Supprimez le compte de stockage Azure.
- Supprimez l’application Azure Function.
- Supprimez le nom du groupe de ressources contenant les ressources.
