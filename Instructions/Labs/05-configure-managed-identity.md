---
lab:
  title: "Configurer une identité managée pour Azure\_SQL\_Database"
  module: Explore Azure SQL Database safety practices for development
---

# Configurer une identité managée pour Azure SQL Database

Dans cet exercice, vous allez ajouter une identité managée à l’exemple d’application web sans stocker d’informations d’identification dans le code.

Azure App Service offre une solution d’hébergement web hautement évolutive et autonome. Une de ses fonctions principales est la mise à disposition d’une identité managée pour votre application, ce qui simplifie la sécurisation de l’accès à Azure SQL Database et à d’autres services Azure. En utilisant des identités managées, vous pouvez améliorer la sécurité de votre application en supprimant la nécessité de stocker des informations sensibles comme les informations d’identification dans des chaînes de connexion. 

Cet exercice devrait prendre environ **30** minutes.

## Avant de commencer

Avant de commencer cet exercice, vous devez avoir :

- Un abonnement Azure avec les autorisations appropriées pour créer et gérer des ressources.
- [**Visual Studio Code**](https://code.visualstudio.com/download?azure-portal=true) installé sur votre ordinateur avec les extensions suivantes installées :
    - [Azure App Service](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azureappservice?azure-portal=true).

## Créer une application Web et une base de données Azure SQL

Pour commencer, nous allons créer une application Web et une base de données Azure SQL.

1. Connectez-vous au [portail Azure](https://portal.azure.com?azure-portal=true).
1. Recherchez et sélectionnez **Abonnements**.
1. Accédez à **Fournisseurs de ressources** sous **Paramètres**, recherchez le fournisseur **Microsoft.Sql**, puis sélectionnez **S’inscrire**.
1. Revenez à la page principale du portail Azure et sélectionnez **Créer une ressource**.
1. Recherchez et sélectionnez **Application web + Base de données**.
1. Sélectionnez **Créer** et renseignez les détails requis :

    | Groupe | Paramètre | Valeur |
    | --- | --- | --- |
    | **Détails du projet** | **Abonnement** | Sélectionnez votre abonnement Azure. |
    | **Détails du projet** | **Groupe de ressources** | Sélectionner ou créer un groupe de ressources |
    | **Détails du projet** | **Région** | Sélectionnez la région dans laquelle vous souhaitez héberger votre application web. |
    | **Détails de l’application web** | **Nom** | Entrez un nom unique pour votre application web |
    | **Détails de l’application web** | **Pile d’exécution** | .NET 8 (LTS) |
    | **Sauvegarde de la base de données** | **Moteur** | SQLAzure |
    | **Sauvegarde de la base de données** | **Nom du serveur** | Entrez un nom unique pour votre serveur SQL |
    | **Sauvegarde de la base de données** | **Nom de la base de données** | Donnez un nom unique à votre base de données |
    | **Hébergement** | **Plan d’hébergement** | De base |

    > **Remarque :** pour les charges de travail de production, sélectionnez **Standard - Applications de production à usage général**. Le nom d’utilisateur et le mot de passe de la nouvelle base de données sont générés automatiquement. Pour récupérer ces valeurs après le déploiement, accédez aux **chaînes de connexion** situées dans la page **Variables d’environnement** de votre application. 

1. Sélectionnez **Vérifier + créer**, puis **Créer**. Le déploiement peut prendre quelques minutes.
1. Connectez-vous à votre base de données dans Azure Data Studio et exécutez le code suivant :

    ```sql
    CREATE TABLE Products (
        ProductID INT PRIMARY KEY,
        ProductName NVARCHAR(100),
        Category NVARCHAR(50),
        Price DECIMAL(10, 2),
        Stock INT
    );
    
    INSERT INTO Products (ProductID, ProductName, Category, Price, Stock) VALUES
    (1, 'Laptop', 'Electronics', 999.99, 50),
    (2, 'Smartphone', 'Electronics', 699.99, 150),
    (3, 'Desk Chair', 'Furniture', 89.99, 200),
    (4, 'Coffee Maker', 'Appliances', 49.99, 100),
    (5, 'Book', 'Books', 19.99, 300);
    ```

## Ajouter un compte en tant qu’administrateur SQL

Ensuite, vous allez ajouter l’accès de votre compte à la base de données. Cette étape est nécessaire, car seuls les comptes authentifiés via Microsoft Entra peuvent créer d’autres utilisateurs de Microsoft Entra ID, qui sont requis pour les étapes suivantes de cet exercice.

1. Accédez au serveur Azure SQL que vous avez créé précédemment.
1. Dans le menu de gauche **Paramètres**, sélectionnez **Microsoft Entra ID**.
1. Sélectionnez **Définir l’administrateur**.
1. Recherchez votre compte et sélectionnez-le.
1. Cliquez sur **Enregistrer**.

## Activer une identité managée

Ensuite, vous allez activer l’identité managée affectée par le système pour votre application web Azure, une bonne pratique de sécurité qui permet la gestion automatisée des informations d’identification.

1. Accédez à votre application web dans le portail Azure.
1. Sous **Paramètres** dans le menu de gauche, sélectionnez **Identité**.
1. Sous l’onglet **Affecté(e) par le système**, définissez **État** sur **Activé**, puis sélectionnez **Enregistrer**. Si vous recevez un message vous demandant si vous souhaitez activer l’identité managée affectée par le système pour votre application web, sélectionnez **Oui**.

## Accorder l’accès à la base de données Azure SQL

1. Connectez-vous à la base de données Azure SQL avec Azure Data Studio Sélectionnez **Microsoft Entra ID - Authentification universelle avec MFA** et indiquez votre nom d’utilisateur.
1. Sélectionnez votre base de données, puis ouvrez un nouvel éditeur de requête.
1. Exécutez les commandes SQL suivantes pour créer un utilisateur pour l’identité managée et attribuer les autorisations nécessaires. Modifiez le script en fournissant le nom de votre application web.

    ```sql
    CREATE USER [your-web-app-name] FROM EXTERNAL PROVIDER;
    ALTER ROLE db_datareader ADD MEMBER [your-web-app-name];
    ALTER ROLE db_datawriter ADD MEMBER [your-web-app-name];
    ```

## Créer une application Web 

Ensuite, vous allez créer une application ASP.NET qui utilise Entity Framework Core avec Azure SQL Database pour afficher une liste de produits provenant de la table de produits.

### Créer votre projet

1. Sur VS Code, créez un dossier. Nommez le dossier de votre projet.
1. Ouvrez le terminal et exécutez la commande suivante pour créer votre projet MVC.
    
    ```dos
        dotnet new mvc
    ```
    Cette étape crée un projet MVC ASP.NET dans le dossier que vous avez choisi et le charge dans Visual Studio Code.

1. Exécutez la commande suivante pour exécuter votre application. 

    ```dos
    dotnet run
    ```
1. Le terminal indique *Now listening on :http://localhost:<port>*. Accédez à l’URL dans votre navigateur web pour accéder à l’application. 

1. Fermez le navigateur web et arrêtez l’application. Vous pouvez également arrêter l’application en appuyant sur`Ctrl+C` dans le terminal VS Code.

### Mettre à jour votre projet pour vous connecter à Azure SQL Database

Ensuite, vous allez mettre à jour certaines configurations qui vous permettront de vous connecter à Azure SQL Database à l’aide de l’identité managée.

1. Dans votre projet, ajoutez les packages NuGet nécessaires pour SQL Server.
    ```dos
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer
    ```
1. Dans le dossier racine de votre projet, ouvrez le fichier **appsettings.json** et insérez la section `ConnectionStrings`. Ici, vous allez remplacer `<server-name>` et `<db-name>` par les noms réels de votre serveur et de votre base de données. Cette chaîne de connexion est utilisée par le constructeur par défaut dans le fichier `Models/MyDbContext.cs` pour établir une connexion à votre base de données.

    ```json
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft.AspNetCore": "Warning"
        }
      },
      "AllowedHosts": "*",
      "ConnectionStrings": {
        "DefaultConnection": "Server=<server-name>.database.windows.net,1433;Initial Catalog=<db-name>;Authentication=Active Directory Default;"
      }
    }
    ```
1. Enregistrez et fermez le fichier.

### Ajouter votre code

1. Dans le dossier **Modèles** de votre projet, créez un fichier **Product.cs** pour votre entité de produit avec le code suivant. Remplacez `<app name>` par le nom réel de votre application.

    ```csharp
    namespace <app name>.Models;
    
    public class Product
    {
        public int ProductId { get; set; }
        public string ProductName { get; set; }
        public string Category { get; set; }
        public decimal Price { get; set; }
        public int Stock { get; set; }
    }
    ```
1. Créez le dossier **Base de données** dans le dossier racine de votre projet.
1. Dans le dossier **Base de données** de votre projet, créez un fichier **MyDbContext.cs** pour votre entité de produit avec le code suivant. Remplacez `<app name>` par le nom réel de votre application.

    ```csharp
    using <app name>.Models;
    
    namespace <app name>.Database;
    
    using Microsoft.EntityFrameworkCore;
    
    public class MyDbContext : DbContext
    {
        public MyDbContext(DbContextOptions<MyDbContext> options) : base(options)
        {
        }
    
        public DbSet<Product> Products { get; set; }
    }    
    ```
1. Dans le dossier **Contrôleurs** de votre projet, modifiez les classes `HomeController` et `IActionResult` pour le fichier **HomeController.cs**, puis ajoutez la variable `_context` avec le code suivant.

    ```csharp
    private MyDbContext _context;

    public HomeController(ILogger<HomeController> logger, MyDbContext context)
    {
        _logger = logger;
        _context = context;
    }

    public IActionResult Index()
    {
        var data = _context.Products.ToList();
        return View(data);
    }
    ```
1. Dans le dossier **Vues -> Accueil** de votre projet, mettez à jour le fichier **Index.cshtml** et ajoutez le code suivant.

    ```html
    <table class="table">
        <thead>
            <tr>
                <th>Product Id</th>
                <th>Product Name</th>
                <th>Category</th>
                <th>Price</th>
                <th>Stock</th>
            </tr>
        </thead>
        <tbody>
            @foreach(var item in Model)
            {
                <tr>
                    <td>@item.ProductId</td>
                    <td>@item.ProductName</td>
                    <td>@item.Category</td>
                    <td>@item.Price</td>
                    <td>@item.Stock</td>
                </tr>
            }
        </tbody>
    </table>
    ```

1. Modifiez le fichier **Program.cs** et insérez l’extrait de code fourni juste au-dessus de la ligne `var app = builder.Build();`. Cette modification garantit que le code s’exécute pendant la séquence de démarrage de l’application. Remplacez `<app name>` par le nom réel de votre application.

    ```csharp
    using Microsoft.EntityFrameworkCore;
    using <app name>.Database;

    var builder = WebApplication.CreateBuilder(args);

    // Add services to the container.
    builder.Services.AddControllersWithViews();
    builder.Services.AddDbContext<MyDbContext>(options =>
        options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
    ```

    > **Remarque :** si vous souhaitez exécuter votre application avant le déploiement, mettez à jour le chaîne de connexion avec les informations d’identification de l’utilisateur SQL. Le nom d’utilisateur et le mot de passe de la base de données ont été générés automatiquement. Pour récupérer ces valeurs après le déploiement, accédez aux **chaînes de connexion** situées dans la page **Variables d’environnement** de votre application. Une fois que vous avez confirmé que l’application s’exécute comme prévu, revenez à l’utilisation de l’identité managée pour un processus de déploiement sécurisé.

### Déploiement de votre code

1. Ouvrez la **palette de commandes** en appuyant sur `Ctrl+Shift+P`.
1. Tapez et sélectionnez **Azure App Service : Déployer sur l’application web**.
1. Sélectionnez le dossier contenant le code de votre application web.
1. Choisissez l’application web créée à l’étape précédente.
    > Remarque : vous pouvez recevoir le message « La configuration requise pour le déploiement est manquante dans votre application ». Sélectionnez **Ajouter une configuration**. Suivez ensuite les instructions et sélectionnez votre abonnement et la ressource App Service.
1. Confirmez le déploiement lorsque vous y êtes invité.

## Tester votre application

Exécutez votre application web et vérifiez qu’elle peut se connecter à Azure SQL Database sans informations d’identification stockées.

1. Ouvrez un navigateur et accédez à l’URL de votre application web Azure (par exemple, https://your-web-app-name.azurewebsites.net)).
1. Vérifiez que votre application web est en cours d’exécution et accessible.
1. Vous devez voir une page web semblable à celle illustrée ci-dessous.

    ![Capture d’écran montrant l’application web après le déploiement.](./Media/01-app-page.png)

## Configurer un déploiement continu (facultatif)

1. Ouvrez la **palette de commandes** en appuyant sur `Ctrl+Shift+P`.
1. Tapez et sélectionnez **Azure App Service : Configurer la livraison continue...**.
1. Suivez les invites pour configurer le déploiement continu à partir de votre dépôt GitHub ou d’Azure DevOps.

Considérez les scénarios où il serait utile d’utiliser une **identité managée affectée par l’utilisateur** au lieu d’une **identité managée affectée par le système**.

## Nettoyer

Lorsque vous travaillez dans votre propre abonnement, il est recommandé, à la fin de chaque projet, de déterminer si vous avez toujours besoin des ressources que vous avez créées. 

Laisser inutilement des ressources en cours d’exécution peut entraîner des coûts supplémentaires. Vous pouvez supprimer les ressources individuellement ou supprimer le groupe de ressources dans le [Portail Azure](https://portal.azure.com?azure-portal=true).

## Plus d’informations

Pour plus d’informations sur les groupes de basculement automatique pour Azure SQL Database, consultez [Identités managées dans Microsoft Entra pour Azure SQL](https://learn.microsoft.com/azure/azure-sql/database/authentication-azure-ad-user-assigned-managed-identity?azure-portal=true).
