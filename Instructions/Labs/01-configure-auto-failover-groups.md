---
lab:
  title: "Activer la résilience des applications avec des groupes de basculement automatique pour Azure\_SQL\_Database"
  module: Get started with Azure SQL Database for cloud-native application development
---

# Activer la résilience des applications avec des groupes de basculement automatique pour Azure SQL Database

Dans cet exercice, vous allez créer deux bases de données Azure SQL qui jouent un rôle primaire et secondaire. Vous allez configurer des groupes de basculement automatique pour veiller à une haute disponibilité et récupération d’urgence des bases de données de votre application et valider l’état de la réplication sur votre application.

Cet exercice devrait prendre environ **30** minutes.

## Avant de commencer

Avant de commencer cet exercice, vous devez avoir :

- Un abonnement Azure avec les autorisations appropriées pour créer et gérer des ressources.
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true) installé sur votre ordinateur avec les extensions suivantes installées :
    - [Kit de développement C#](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit?azure-portal=true).

## Créer des serveurs Azure SQL principaux et secondaires

Tout d’abord, nous allons configurer les serveurs principaux et secondaires, et nous allons utiliser l’exemple de base de données **AdventureWorksLT**.

1. Connectez-vous au [portail Azure](https://portal.azure.com?azure-portal=true).

1. Sélectionnez l’icône Cloud Shell dans le coin supérieur droit du portail Azure. Il ressemble au symbole `>_`. Si vous y êtes invité, choisissez **Bash** comme type d’interpréteur de commandes.

1. Exécutez les commandes suivantes dans le terminal Cloud Shell. Remplacez les valeurs `<your_resource_group>`, `<your_primary_server>`, `<your_location>`, `<your_secondary_server>` et `<your_admin_password>` par vos valeurs réelles :

    * Créer un groupe de ressources
    ```azurecli
    az group create --name <your_resource_group> --location <your_location>
    ```

    * Créer le serveur SQL principal
    ```azurecli        
    az sql server create --name <your_primary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>
    ```

    * Créer le serveur SQL secondaire Même script, modifier uniquement le nom et l’emplacement du serveur
    ```azurecli
    az sql server create --name <your_secondary_server> --resource-group <your_resource_group> --location <your_location> --admin-user "sqladmin" --admin-password <your_admin_password>    
    ```

    * Créer un exemple de base de données sur le serveur principal avec le niveau tarifaire spécifié
    ```azurecli
    az sql db create --resource-group <your_resource_group> --server <your_primary_server> --name AdventureWorksLT --sample-name AdventureWorksLT --service-objective "S0"    
    ```
    
1. Une fois les déploiements terminés, accédez au serveur Azure SQL principal que vous avez créé.
1. Dans le volet de navigation gauche, sous **Sécurité**, sélectionnez **Mise en réseau**. Ajoutez votre adresse IP aux règles de pare-feu.
1. Sélectionnez l’option **Autoriser les services et les ressources Azure à accéder à ce serveur**.
1. Cliquez sur **Enregistrer**.
1. Répétez les étapes ci-dessus pour le deuxième serveur.

    Ces étapes garantissent que vous disposez d’un environnement Azure SQL Database structuré et redondant prêt à être utilisé.

## Configurer des groupes de basculement automatique

Ensuite, vous allez créer un groupe de basculement automatique pour la base de données Azure SQL que vous avez configurée précédemment. Pour cela, vous devez établir un groupe de basculement entre deux serveurs et vérifier l’installation pour vous assurer qu’elle fonctionne correctement.

1. Exécutez les commandes suivantes dans le terminal Cloud Shell. Remplacez les valeurs `<your_failover_group>`, `<your_resource_group>`, `<your_primary_server>` et `<your_secondary_server>` par vos valeurs réelles :

    * Créer le groupe de basculement
    ```azurecli
    az sql failover-group create -n <your_failover_group> -g <your_resource_group> -s <your_primary_server> --partner-server <your_secondary_server> --failover-policy Automatic --grace-period 1 --add-db AdventureWorksLT
    ```

    * Vérifier le groupe de basculement
    ```azurecli    
    az sql failover-group show -n <your_failover_group> -g <your_resource_group> -s <your_primary_server>
    ```

    > Prenez un moment pour passer en revue les résultats et les valeurs `partnerServers`. Pourquoi est-ce important ?

    > En vérifiant l’attribut `role` au sein de chaque serveur partenaire, vous pouvez déterminer si un serveur agit actuellement en tant que serveur principal ou secondaire. Ces informations sont essentielles pour comprendre la configuration et la préparation actuelles du groupe de basculement. Elles vous aident à évaluer l’impact potentiel sur votre application pendant les scénarios de basculement et garantit que votre configuration est correctement définie pour la haute disponibilité et la récupération d’urgence.
    
## Intégrer avec le code de l’application

Pour connecter votre application .NET au point de terminaison Azure SQL Database, vous devez suivre ces étapes.

1. Dans Visual Studio Code, ouvrez le terminal et exécutez les commandes suivantes pour installer le package `Microsoft.Data.SqlClient` et créer une application console .NET.

    ```bash
    dotnet new console -n AdventureWorksLTApp
    cd AdventureWorksLTApp 
    dotnet add package Microsoft.Data.SqlClient --version 5.2.1
    ```

1. Ouvrez le dossier `AdventureWorksLTApp` créé à l’étape précédente dans **Visual Studio Code**.

1. Créez un fichier `appsettings.json` dans le répertoire racine de votre projet. Ce fichier de configuration stocke la chaîne de connexion de votre base de données. Veillez à remplacer les valeurs `<your_failover_group>` et `<your_password>` de la chaîne de connexion par vos détails réels.

    ```json
    {
      "ConnectionStrings": {
        "FailoverGroupConnection": "Server=tcp:<your_failover_group>.database.windows.net,1433;Initial Catalog=AdventureWorksLT;Persist Security Info=False;User ID=sqladmin;Password=<your_password>;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
      }
    }
    ```

1. Ouvrez le fichier `.csproj` dans **Visual Studio Code** et ajoutez le contenu suivant juste sous la balise `</PropertyGroup>`.

    ```xml
    <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
    </ItemGroup>
    ```

    Votre fichier `.csproj` complet doit ressembler à ceci :

    ```xml
    <Project Sdk="Microsoft.NET.Sdk">

      <PropertyGroup>
        <OutputType>Exe</OutputType>
        <TargetFramework>net8.0</TargetFramework>
        <ImplicitUsings>enable</ImplicitUsings>
        <Nullable>enable</Nullable>
      </PropertyGroup>
    
      <ItemGroup>
        <PackageReference Include="Microsoft.Data.SqlClient" Version="5.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration" Version="6.0.0" />
        <PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="6.0.0" />
      </ItemGroup>
    
    </Project>
    ```

1. Ouvrez le fichier `Program.cs` dans **Visual Studio Code**. Dans l’éditeur, remplacez tout le code existant par le code fourni ci-dessous.

    > **Remarque :** prenez un moment pour passer en revue le code et observer comment il imprime des informations sur les serveurs principaux et secondaires dans le groupe de basculement automatique.

    ```csharp
    using System;
    using Microsoft.Data.SqlClient;
    using Microsoft.Extensions.Configuration;
    using System.IO;
    
    namespace AdventureWorksLTApp
    {
        class Program
        {
            static void Main(string[] args)
            {
                var configuration = new ConfigurationBuilder()
                    .SetBasePath(Directory.GetCurrentDirectory())
                    .AddJsonFile("appsettings.json")
                    .Build();
    
                string connectionString = configuration.GetConnectionString("FailoverGroupConnection");
    
                ExecuteQuery(connectionString);
            }
    
            static void ExecuteQuery(string connectionString)
            {
                using (SqlConnection connection = new SqlConnection(connectionString))
                {
                    try
                    {
                        connection.Open();
                        string query = @"
                            SELECT 
                                @@SERVERNAME AS [Primary_Server],
                                partner_server AS [Secondary_Server],
                                partner_database AS [Database],
                                replication_state_desc
                            FROM 
                                sys.dm_geo_replication_link_status";
    
                        using (SqlCommand command = new SqlCommand(query, connection))
                        {
                            using (SqlDataReader reader = command.ExecuteReader())
                            {
                                while (reader.Read())
                                {
                                    Console.WriteLine($"Primary Server: {reader["Primary_Server"]}");
                                    Console.WriteLine($"Secondary Server: {reader["Secondary_Server"]}");
                                    Console.WriteLine($"Database: {reader["Database"]}");
                                    Console.WriteLine($"Replication State: {reader["replication_state_desc"]}");
                                }
                            }
                        }
                    }
                    catch (Exception ex)
                    {
                        Console.WriteLine($"Error executing query: {ex.Message}");
                    }
                    finally
                    {
                        connection.Close();
                    }
                }
            }
        }
    }
    ```

1. Exécutez le code en sélectionnant **Exécuter** > **Démarrer le débogage** dans le menu, ou appuyez simplement sur **F5**. Vous pouvez également démarrer l’application en sélectionnant le bouton de la lecture dans la barre d’outils supérieur.

    > **Important :** si vous recevez un message *« Vous n’avez pas d’extension pour le débogage C#. Chercher une extension C# dans MarketPlace ? »*, vérifiez que l’extension **Kit de développement C#** est installée.

1. Après avoir exécuté le code, vous devez voir la sortie dans l’onglet **Console de débogage** de Visual Studio Code.

    ```
    Primary Server: <your_server_name>
    Secondary Server: <your_server_name>
    Database: AdventureWorksLT
    Replication State: CATCH_UP
    ```
    
    L’état de réplication `CATCH_UP` signifie que la base de données est entièrement synchronisée avec son partenaire et qu’elle est prête pour le basculement. La surveillance de l’état de réplication peut aider à identifier les goulots d’étranglement des performances et à s’assurer que la réplication des données se produit efficacement.

## Basculer vers une région secondaire

Imaginez un scénario où la base de données Azure SQL primaire rencontre des problèmes en raison d’une panne régionale. Pour maintenir la continuité du service et réduire les temps d’arrêt, vous devez basculer votre application vers le réplica secondaire en effectuant un basculement forcé.

Lors d’un basculement forcé, toutes les nouvelles sessions TDS sont automatiquement routées vers le serveur secondaire, qui devient alors le serveur principal. Le plus intéressant, c’est que vous n’avez pas besoin de modifier la chaîne de connexion de l’application, car le point de terminaison reste le même.

Nous allons lancer un basculement et exécuter notre application pour vérifier l’état de nos serveurs principal et secondaire.

1. Revenez au portail Azure et ouvrez une nouvelle instance du terminal Cloud Shell. Exécutez le code suivant. Remplacez les valeurs `<your_failover_group>`, `<your_resource_group>` et `<your_primary_server>` par vos valeurs réelles. La valeur du paramètre `--server` doit être le secondaire actuel.

    ```azurecli
    az sql failover-group set-primary --name <your_failover_group> --resource-group <your_resource_group> --server <your_server_name>
    ```

    > **Remarque** : cette opération peut prendre quelques minutes.

1. Une fois le basculement terminé, réexécutez l’application pour vérifier l’état de la réplication. Vous devez voir que le serveur secondaire a maintenant pris le relais comme serveur principal et que le serveur principal d’origine est devenu le serveur secondaire.

Tenez compte des raisons pour lesquelles vous souhaitez placer vos bases de données d’application primaires et secondaires dans la même région, et quand il peut être utile de choisir différentes régions.

## Nettoyer

Lorsque vous travaillez dans votre propre abonnement, il est recommandé, à la fin de chaque projet, de déterminer si vous avez toujours besoin des ressources que vous avez créées. 

Laisser inutilement des ressources en cours d’exécution peut entraîner des coûts supplémentaires. Vous pouvez supprimer les ressources individuellement ou supprimer le groupe de ressources dans le [Portail Azure](https://portal.azure.com?azure-portal=true).

## Plus d’informations

Pour plus d’informations sur les groupes de basculement automatique pour Azure SQL Database, consultez [Vue d’ensemble des groupes de basculement et meilleures pratiques (Base de données Azure SQL)](https://learn.microsoft.com/azure/azure-sql/database/failover-group-sql-db?azure-portal=true).
