# M3-Industrial-IT - Gestion de Chariot Industriel

## 📝 Présentation du projet
Ce répertoire contient le travail collaboratif sur le module 3, portant sur la conception et le développement d'un système de gestion de chariot industriel. Le projet intègre une base de données MySQL, une interface de contrôle en C# (Windows Forms) et une logique de commande sur automate Codesys.

## 📍 Sommaire
1. [Étape 1 : Création de la base de données (MySQL)](#étape-1--création-de-la-base-de-données-mysql)
2. [Étape 2 : Développement de l'application C# (Windows Forms)](#étape-2--développement-de-lapplication-c-windows-forms)
3. [Étape 3 : Développement de l'automate (Codesys)](#étape-3--développement-de-lautomate-codesys)
4. [Étape 4 : Communication entre C# et Codesys](#étape-4--communication-entre-c-et-codesys)
5. [Étape 5 : Tests et Validation](#étape-5--tests-et-validation)
6. [Étape 6 : Documentation et Livrables](#étape-6--documentation-et-livrables)
7. [Journal de travail](#journal-de-travail)
8. [Checklist Finale](#checklist-finale)
9. [Conseils](#conseils)

---

## 🗄️ ÉTAPE 1 : CRÉATION DE LA BASE DE DONNÉES (MySQL)

### 1.1. Script SQL pour créer les tables
À partir du MCD, voici le script SQL pour créer la base de données :

```sql
-- Création de la base de données
CREATE DATABASE IF NOT EXISTS GestionChariot;
USE GestionChariot;

-- Table des états (pour les lots)
CREATE TABLE Etat (
    Id_Etat INT AUTO_INCREMENT PRIMARY KEY,
    Libelle VARCHAR(50) NOT NULL
);

-- Insertion des états possibles
INSERT INTO Etat (Libelle) VALUES
('En attente'), ('En cours'), ('Terminé'), ('Erreur');

-- Table des recettes
CREATE TABLE Recette (
    Id_Recette INT AUTO_INCREMENT PRIMARY KEY,
    Nom VARCHAR(50) NOT NULL UNIQUE,
    Date_Heure_creation DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Table des opérations (liées aux recettes)
CREATE TABLE Operation (
    Id_Operation INT AUTO_INCREMENT PRIMARY KEY,
    Id_Recette INT NOT NULL,
    Nom VARCHAR(50) NOT NULL,
    PositionMoteur INT NOT NULL,
    Temps_attente INT NOT NULL COMMENT 'En secondes (ignoré si Cycle_verin = true)',
    Cycle_verin BOOLEAN NOT NULL DEFAULT FALSE,
    Quittance BOOLEAN NOT NULL DEFAULT FALSE,
    Sens_moteur BOOLEAN NOT NULL DEFAULT FALSE,
    FOREIGN KEY (Id_Recette) REFERENCES Recette(Id_Recette) ON DELETE CASCADE,
    UNIQUE KEY (Id_Recette, PositionMoteur) -- Une position par recette
);

-- Table des lots
CREATE TABLE Lot (
    Id_Lot INT AUTO_INCREMENT PRIMARY KEY,
    Nom VARCHAR(50) NOT NULL UNIQUE,
    Quantite INT NOT NULL,
    Id_Recette INT NOT NULL,
    Id_Etat INT NOT NULL DEFAULT 1, -- Par défaut "En attente"
    Date_Heure_creation DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (Id_Recette) REFERENCES Recette(Id_Recette),
    FOREIGN KEY (Id_Etat) REFERENCES Etat(Id_Etat)
);

-- Table des événements (traçabilité)
CREATE TABLE Evenement (
    Id_Evenement INT AUTO_INCREMENT PRIMARY KEY,
    Id_Lot INT NOT NULL,
    Message VARCHAR(50) NOT NULL,
    Date_Heure_message DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (Id_Lot) REFERENCES Lot(Id_Lot) ON DELETE CASCADE
);
```

### 1.2. Contraintes à respecter

- **Cycle vérin :**
  - Si `Cycle_verin = true` → Le `Temps_attente` est ignoré (mis à 0).
  - Si `Cycle_verin = false` → Le `Temps_attente` doit être renseigné.

- **Quittance :**
  - Si `Quittance = true` → L’opérateur doit valider manuellement l’opération avant de passer à la suivante.

---

## 💻 ÉTAPE 2 : DÉVELOPPEMENT DE L'APPLICATION C# (Windows Forms)

### 2.1. Structure du projet
Créez un projet Windows Forms dans Visual Studio avec les classes suivantes :

```
GestionChariot/
├── Models/          # Classes C# correspondant aux tables SQL
│   ├── Recette.cs
│   ├── Operation.cs
│   ├── Lot.cs
│   ├── Etat.cs
│   └── Evenement.cs
├── Services/        # Logique métier et accès à la BDD
│   ├── RecetteService.cs
│   ├── LotService.cs
│   ├── EvenementService.cs
│   └── DatabaseService.cs
└── Forms/           # Interfaces graphiques
    ├── MainForm.cs          # Menu principal
    ├── RecetteForm.cs       # Éditeur de recettes
    ├── LotForm.cs           # Éditeur de lots
    └── TraçabiliteForm.cs   # Visualisation des événements
```

### 2.2. Exemple de code pour les modèles

📄 **Models/Recette.cs**
```csharp
public class Recette
{
    public int Id_Recette { get; set; }
    public string Nom { get; set; }
    public DateTime Date_Heure_creation { get; set; }
    public List<Operation> Operations { get; set; } = new List<Operation>();
}
```

📄 **Models/Operation.cs**
```csharp
public class Operation
{
    public int Id_Operation { get; set; }
    public int Id_Recette { get; set; }
    public string Nom { get; set; }
    public int PositionMoteur { get; set; }
    public int Temps_attente { get; set; }
    public bool Cycle_verin { get; set; }
    public bool Quittance { get; set; }
    public bool Sens_moteur { get; set; } // false = avant, true = arrière
}
```

### 2.3. Connexion à la base de données

📄 **Services/DatabaseService.cs**
```csharp
using MySql.Data.MySqlClient;

public class DatabaseService
{
    private readonly string _connectionString = "Server=localhost;Database=GestionChariot;Uid=root;Pwd=;";

    public MySqlConnection GetConnection()
    {
        return new MySqlConnection(_connectionString);
    }
}
```

📄 **Services/RecetteService.cs**
```csharp
public class RecetteService
{
    private readonly DatabaseService _dbService;

    public RecetteService(DatabaseService dbService)
    {
        _dbService = dbService;
    }

    public List<Recette> GetAllRecettes()
    {
        var recettes = new List<Recette>();
        using (var conn = _dbService.GetConnection())
        {
            conn.Open();
            var cmd = new MySqlCommand("SELECT * FROM Recette", conn);
            using (var reader = cmd.ExecuteReader())
            {
                while (reader.Read())
                {
                    recettes.Add(new Recette
                    {
                        Id_Recette = reader.GetInt32("Id_Recette"),
                        Nom = reader.GetString("Nom"),
                        Date_Heure_creation = reader.GetDateTime("Date_Heure_creation")
                    });
                }
            }
        }
        return recettes;
    }

    public void AddRecette(Recette recette)
    {
        using (var conn = _dbService.GetConnection())
        {
            conn.Open();
            var cmd = new MySqlCommand(
                "INSERT INTO Recette (Nom) VALUES (@Nom)", conn);
            cmd.Parameters.AddWithValue("@Nom", recette.Nom);
            cmd.ExecuteNonQuery();
        }
    }

    public void AddOperation(Operation operation)
    {
        using (var conn = _dbService.GetConnection())
        {
            conn.Open();
            var cmd = new MySqlCommand(
                @"INSERT INTO Operation
                (Id_Recette, Nom, PositionMoteur, Temps_attente, Cycle_verin, Quittance, Sens_moteur)
                VALUES (@Id_Recette, @Nom, @PositionMoteur, @Temps_attente, @Cycle_verin, @Quittance, @Sens_moteur)", conn);
            cmd.Parameters.AddWithValue("@Id_Recette", operation.Id_Recette);
            cmd.Parameters.AddWithValue("@Nom", operation.Nom);
            cmd.Parameters.AddWithValue("@PositionMoteur", operation.PositionMoteur);
            cmd.Parameters.AddWithValue("@Temps_attente", operation.Cycle_verin ? 0 : operation.Temps_attente);
            cmd.Parameters.AddWithValue("@Cycle_verin", operation.Cycle_verin);
            cmd.Parameters.AddWithValue("@Quittance", operation.Quittance);
            cmd.Parameters.AddWithValue("@Sens_moteur", operation.Sens_moteur);
            cmd.ExecuteNonQuery();
        }
    }
}
```

### 2.4. Interface graphique (Windows Forms)

📄 **Forms/RecetteForm.cs**
```csharp
public partial class RecetteForm : Form
{
    private readonly RecetteService _recetteService;
    private Recette _currentRecette;

    public RecetteForm()
    {
        InitializeComponent();
        _recetteService = new RecetteService(new DatabaseService());
        LoadRecettes();
    }

    private void LoadRecettes()
    {
        var recettes = _recetteService.GetAllRecettes();
        listBoxRecettes.DataSource = recettes;
        listBoxRecettes.DisplayMember = "Nom";
    }

    private void btnNouvelleRecette_Click(object sender, EventArgs e)
    {
        var nomRecette = txtNomRecette.Text.Trim();
        if (string.IsNullOrEmpty(nomRecette))
        {
            MessageBox.Show("Le nom de la recette est obligatoire.");
            return;
        }

        _currentRecette = new Recette { Nom = nomRecette };
        _recetteService.AddRecette(_currentRecette);
        LoadRecettes();
    }

    private void btnAjouterOperation_Click(object sender, EventArgs e)
    {
        if (_currentRecette == null)
        {
            MessageBox.Show("Veuillez d'abord créer une recette.");
            return;
        }

        var operation = new Operation
        {
            Id_Recette = _currentRecette.Id_Recette,
            Nom = txtNomOperation.Text,
            PositionMoteur = (int)numericPosition.Value,
            Temps_attente = (int)numericTempsAttente.Value,
            Cycle_verin = checkCycleVerin.Checked,
            Quittance = checkQuittance.Checked,
            Sens_moteur = checkSensMoteur.Checked
        };

        _recetteService.AddOperation(operation);
        MessageBox.Show("Opération ajoutée avec succès !");
    }
}
```

📄 **Forms/LotForm.cs**
```csharp
public partial class LotForm : Form
{
    private readonly LotService _lotService;

    public LotForm()
    {
        InitializeComponent();
        _lotService = new LotService(new DatabaseService());
        LoadRecettes();
        LoadLots();
    }

    private void LoadRecettes()
    {
        var recettes = _lotService.GetAllRecettes();
        comboBoxRecettes.DataSource = recettes;
        comboBoxRecettes.DisplayMember = "Nom";
    }

    private void LoadLots()
    {
        var lots = _lotService.GetAllLots();
        dataGridViewLots.DataSource = lots;
    }

    private void btnCreerLot_Click(object sender, EventArgs e)
    {
        var selectedRecette = (Recette)comboBoxRecettes.SelectedItem;
        var quantite = (int)numericQuantite.Value;
        var nomLot = txtNomLot.Text.Trim();

        if (string.IsNullOrEmpty(nomLot))
        {
            MessageBox.Show("Le nom du lot est obligatoire.");
            return;
        }

        _lotService.AddLot(new Lot
        {
            Nom = nomLot,
            Quantite = quantite,
            Id_Recette = selectedRecette.Id_Recette
        });

        LoadLots();
    }
}
```

---

## 🤖 ÉTAPE 3 : DÉVELOPPEMENT DE L'AUTOMATE (CODESYS)

### 3.1. Communication avec MySQL depuis Codesys
Codesys ne supporte pas nativement MySQL, mais vous pouvez utiliser :
- Un serveur OPC UA pour faire le lien entre Codesys et C#.
- Un script Python qui lit/écrit dans MySQL et communique avec Codesys via Modbus TCP.

**Exemple de structure Codesys**
```
GestionChariot (Codesys Project)
├── PLC_PRG (Programme principal)
│   ├── VAR
│   │   ├── currentLot : Lot;          // Lot en cours de production
│   │   ├── currentOperation : INT;    // Opération en cours
│   │   ├── isRunning : BOOL;          // Production en cours
│   │   └── alarmTriggered : BOOL;     // Alarme active
│   └── Logic
│       ├── ReadLotFromDB();           // Lit le prochain lot à produire
│       ├── ExecuteOperation();        // Exécute une opération
│       └── LogEvent();                // Enregistre un événement dans la BDD
└── Communication
    ├── ModbusTCP_Server;              // Pour communiquer avec un script Python
    └── OPCUA_Server;                  // Alternative
```

### 3.2. Exemple de code Codesys (IEC 61131-3 ST)

```iecst
PROGRAM PLC_PRG
VAR
    currentLot : Lot;
    currentOperation : INT := 1;
    isRunning : BOOL := FALSE;
    alarmTriggered : BOOL := FALSE;
    timerOperation : TON; // Timer pour le temps d'attente
END_VAR

// Lecture du prochain lot à produire (via OPC UA ou Modbus)
IF NOT isRunning THEN
    currentLot := ReadLotFromDB();
    IF currentLot.Id_Lot > 0 THEN
        isRunning := TRUE;
        LogEvent(currentLot.Id_Lot, 'Début du lot ' + currentLot.Nom);
    END_IF
END_IF

// Exécution des opérations
IF isRunning THEN
    // Récupérer l'opération courante depuis la BDD
    VAR_OPERATION := GetOperation(currentLot.Id_Recette, currentOperation);

    // Exécuter l'opération
    IF NOT timerOperation.Q THEN
        ExecuteOperation(VAR_OPERATION);
        timerOperation(IN := TRUE, PT := T#VAR_OPERATION.Temps_attente * 1000);
    ELSE
        timerOperation(IN := FALSE);
        IF timerOperation.Q THEN
            // Passer à l'opération suivante
            currentOperation := currentOperation + 1;
            timerOperation(IN := FALSE);

            // Vérifier si le lot est terminé
            IF currentOperation > GetOperationCount(currentLot.Id_Recette) THEN
                LogEvent(currentLot.Id_Lot, 'Fin du lot ' + currentLot.Nom);
                UpdateLotState(currentLot.Id_Lot, 3); // 3 = Terminé
                isRunning := FALSE;
                currentOperation := 1;
            END_IF
        END_IF
    END_IF

    // Gestion des alarmes (ex : barrière lumineuse coupée)
    IF alarmTriggered THEN
        LogEvent(currentLot.Id_Lot, 'Alarme : Barrière lumineuse coupée');
        UpdateLotState(currentLot.Id_Lot, 4); // 4 = Erreur
        isRunning := FALSE;
    END_IF
END_IF
```

---

## 🔌 ÉTAPE 4 : COMMUNICATION ENTRE C# ET CODESYS

### Option 1 : OPC UA (Recommandé)

1. **Dans Codesys :**
   - Activez le serveur OPC UA (Projet → Communication → OPC UA).
   - Déclarez les variables à exposer (ex : `currentLot`, `isRunning`).

2. **Dans C# :**
   - Utilisez la bibliothèque `OPCFoundation/UA-.NET` pour lire/écrire les variables Codesys.

```csharp
using Opc.Ua;
using Opc.Ua.Client;

public class OpcUaService
{
    private ApplicationConfiguration _config;
    private Session _session;

    public async Task Connect(string endpointUrl)
    {
        _config = new ApplicationConfiguration
        {
            ApplicationName = "GestionChariotClient",
            ApplicationType = ApplicationType.Client,
            SecurityConfiguration = new SecurityConfiguration
            {
                ApplicationCertificate = new CertificateIdentifier { StoreType = "X509Store" },
                TrustedPeerCertificates = new CertificateTrustList { StoreType = "Directory" }
            }
        };

        var endpoint = CoreClientUtils.SelectEndpoint(endpointUrl, false);
        _session = await Session.Create(
            _config, endpoint, false, false, _config.ApplicationName, 60000, null, null);
    }

    public void WriteVariable(string nodeId, object value)
    {
        var node = new NodeId(nodeId);
        _session.Write(null, new WriteValue
        {
            NodeId = node,
            AttributeId = Attributes.Value,
            Value = new DataValue { Value = value }
        });
    }
}
```

### Option 2 : Modbus TCP + Script Python

1. **Dans Codesys :**
   - Configurez un serveur Modbus TCP (Projet → Communication → Modbus TCP Server).
   - Mappez les variables à des registres Modbus.

2. **Script Python (intermédiaire) :**

```python
from pymodbus.client import ModbusTcpClient
import mysql.connector

# Connexion à Codesys (Modbus TCP)
client = ModbusTcpClient('192.168.1.1')  # IP de l'automate

# Connexion à MySQL
db = mysql.connector.connect(
    host="localhost",
    user="root",
    password="",
    database="GestionChariot"
)

def read_lot_from_db():
    cursor = db.cursor()
    cursor.execute("SELECT * FROM Lot WHERE Id_Etat = 1 LIMIT 1")  # 1 = En attente
    return cursor.fetchone()

def update_lot_state(lot_id, state_id):
    cursor = db.cursor()
    cursor.execute("UPDATE Lot SET Id_Etat = %s WHERE Id_Lot = %s", (state_id, lot_id))
    db.commit()

# Boucle principale
while True:
    lot = read_lot_from_db()
    if lot:
        # Écrire le lot dans Codesys via Modbus
        client.write_register(0, lot[0])  # Id_Lot
        client.write_register(1, lot[3])  # Id_Recette
        update_lot_state(lot[0], 2)      # 2 = En cours
```

---

## 🧪 ÉTAPE 5 : TESTS ET VALIDATION

### 5.1. Tests unitaires (C#)
Testez chaque méthode des Services avec des données fictives.

📄 **Exemple pour RecetteService :**
```csharp
[TestMethod]
public void AddRecette_ShouldInsertIntoDatabase()
{
    var dbService = new DatabaseService();
    var recetteService = new RecetteService(dbService);
    var recette = new Recette { Nom = "TestRecette" };

    recetteService.AddRecette(recette);

    var recettes = recetteService.GetAllRecettes();
    Assert.IsTrue(recettes.Any(r => r.Nom == "TestRecette"));
}
```

### 5.2. Tests d'intégration (C# ↔ MySQL ↔ Codesys)
1. Créez un lot dans l’application C#.
2. Vérifiez qu’il apparaît dans la base de données.
3. Lancez la production dans Codesys et vérifiez que :
   - Le lot passe en état "En cours".
   - Les événements sont enregistrés dans `Evenement`.
   - Le lot passe en état "Terminé" à la fin.

### 5.3. Gestion des erreurs
- **Alarmes :** Simulez une coupure de barrière lumineuse dans Codesys et vérifiez que le lot passe en "Erreur".
- **Quittance manuelle :** Testez le cas où une opération nécessite une validation de l’opérateur.

---

## 📚 ÉTAPE 6 : DOCUMENTATION ET LIVRABLES

### 6.1. Cahier des charges (orienté client)
- **Description du projet :** Objectifs, acteurs, processus métier.
- **Fonctionnalités :** Éditeur de lots/recettes (C#), Traçabilité, Gestionnaire de production (Codesys).
- **Maquettes :** Ajoutez des captures d’écran des interfaces C# et Codesys.
- **Base de données :** MCD + dictionnaire de données.

### 6.2. Documentation technique
- **Architecture :** Diagramme de déploiement (C# ↔ MySQL ↔ Codesys).
- **Code :** Commentaires dans le code, explications des algorithmes critiques.
- **Manuel utilisateur :** Guide pour créer un lot/recette et lancer une production.

---

## 🗓️ Journal de travail

| Date | Heure début | Heure fin | Activité | Catégorie | Durée |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 20.03.2026 | 08:30 | 10:00 | Création du MCD | Conception BDD | 1.5h |
| 21.03.2026 | 13:00 | 15:30 | Développement RecetteService (C#) | Développement | 2.5h |

---

## 📋 CHECKLIST FINALE

| Tâche | Statut |
| :--- | :---: |
| Base de données créée (MySQL) | ⬜ |
| MCD et dictionnaire de données validés | ⬜ |
| Application C# (Windows Forms) fonctionnelle | ⬜ |
| Automate Codesys opérationnel | ⬜ |
| Communication C# ↔ Codesys (OPC UA/Modbus) | ⬜ |
| Tests unitaires et d'intégration passés | ⬜ |
| Documentation complète | ⬜ |
| Journal de travail à jour | ⬜ |

---

## 💡 Conseils // BDD First

1. **Commencez par la base de données :** Une BDD bien conçue simplifie tout le reste.
2. **Développez par itérations :** D’abord la partie C# (éditeur de lots/recettes), puis Codesys, puis la communication.
3. **Testez tôt :** Ne laissez pas les tests pour la fin. Testez chaque fonctionnalité dès qu’elle est codée.
4. **Utilisez Git :** Versionnez votre code pour éviter les pertes (GitHub/GitLab).
5. **Demandez des feedbacks :** Montrez vos maquettes et votre code aux enseignants pour avoir des retours.
