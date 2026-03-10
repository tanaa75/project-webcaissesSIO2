# Compte Rendu Technique — BTS SIO

---

## Partie 1 — Bases de données & Sécurité

### 1.1 — Schéma relationnel

Le schéma relationnel proposé est le suivant :

| Entité | Attributs |
|---|---|
| **Formule** | `id`, `libelle`, `prix`, `nbVendeursSimultanes` |
| **Client** | `id`, `raisonSociale`, `rue`, `codePostal`, `ville`, `telephone`, `#idFormule` |
| **Point de vente** | `id`, `formule` |
| **Programme de fidélisation** | `tampon`, `montantAchat`, `points` |
| **Choix des langues des modules** | `(...)` |

---

### 1.2 — Faille de sécurité : envoi du mot de passe en clair

L'actuelle procédure d'envoi de mot de passe constitue une **faille critique** : le mot de passe est transmis en clair, ce qui ne respecte pas les critères du **DICP**, notamment le principe de **Confidentialité**.

**Solution proposée :**
- Lors de la première connexion, envoyer un **e-mail contenant un lien avec un identifiant temporaire**.
- Laisser ensuite l'utilisateur **définir lui-même son identifiant et son mot de passe**.

---

### 1.3 — Problème : absence de gestion de l'historique

La table actuelle ne permet pas de gérer l'historique nécessaire à la **facturation mensuelle**, car des données essentielles sont manquantes, notamment les **dates**.

---

### 1.4 — Correction : ajout de données temporelles

Il faut **ajouter des champs de date** afin de gérer correctement l'historique de la facturation mensuelle.

---

### 1.5 — Trigger SQL : audit des changements de formule

Le trigger ci-dessous enregistre tout changement de formule sur un point de vente dans une table d'audit.

```sql
CREATE TRIGGER tg_audit_changement_formule
AFTER UPDATE ON PointDeVente
FOR EACH ROW
BEGIN
    IF OLD.idFormule <> NEW.idFormule THEN
        INSERT INTO AuditFormule (idPointDeVente, ancien_idFormule, nouveau_idFormule, date_changement)
        VALUES (OLD.id, OLD.idFormule, NEW.idFormule, NOW());
    END IF;
END;
```

---

## Partie 2 — Requêtes SQL & Sécurité applicative

### 2.1 — Requêtes SQL

**a) Consommateurs ayant effectué au moins un achat en 2017**

```sql
SELECT DISTINCT nom, prenom, mail
FROM Conso
JOIN Vente ON Conso.id = Vente.idConso
WHERE YEAR(dateVente) = 2017;
```

**b) Nombre de consommateurs fidèles âgés de 18 à 30 ans**

```sql
SELECT COUNT(*) AS nb_conso_18_30
FROM ConsoFidele
WHERE TIMESTAMPDIFF(YEAR, dateNaiss, CURDATE()) BETWEEN 18 AND 30;
```

**c) Total des ventes par consommateur**

```sql
SELECT nom, prenom, SUM(montantVente) AS total_ventes
FROM Conso
LEFT JOIN Vente ON Conso.id = Vente.idConso
GROUP BY Conso.id, nom, prenom;
```

---

### 2.2 — Injection SQL : analyse de la faille

En concaténant directement la variable `seuilVentes` (issue d'une saisie utilisateur) dans la chaîne SQL, on permet à un attaquant de **modifier la structure de la requête**.

> **Exemple d'attaque :** si l'utilisateur saisit `1; DROP TABLE Conso;`, la requête exécutée pourrait **supprimer toute la table**. Des variables non sécurisées permettent également de contourner l'authentification ou d'accéder à des données confidentielles.

---

### 2.3 — Correction : utilisation des `PreparedStatement`

```java
public static ArrayList<Conso> listeConsoAFideliser(int seuilVentes, String dateDeb, String dateFin) {
    ArrayList<Conso> lesConsos = new ArrayList<Conso>();

    // Requête SQL avec des marqueurs de paramètres (?)
    String requete = "SELECT nom, prenom, tel, mail " +
                     "FROM Conso " +
                     "JOIN Vente ON Conso.id = Vente.idConso " +
                     "WHERE Conso.id NOT IN (SELECT id FROM ConsoFidele) " +
                     "AND dateVente BETWEEN ? AND ? " +
                     "GROUP BY Conso.id, nom, prenom, tel, mail " +
                     "HAVING COUNT(*) >= ?";

    try {
        Connection dbConnect = DriverManager.getConnection("jdbc:mysql:" + dbURL, adminUser, adminPwd);

        // PreparedStatement : protection contre l'injection SQL
        PreparedStatement pstmt = dbConnect.prepareStatement(requete);
        pstmt.setString(1, dateDeb);
        pstmt.setString(2, dateFin);
        pstmt.setInt(3, seuilVentes);

        ResultSet res = pstmt.executeQuery();

        while (res.next()) {
            // Colonnes : 1=nom, 2=prenom, 3=tel, 4=mail
            Conso leConso = new Conso(res.getString(1), res.getString(2), res.getString(4), res.getString(3));
            lesConsos.add(leConso);
        }

        res.close();
        pstmt.close();
        dbConnect.close();

    } catch (SQLException e) {
        e.printStackTrace();
    }

    return lesConsos;
}
```

---

### 2.4 — Test unitaire : initialisation des points

```java
public void testInitConso() throws Exception {
    ConsoFidele consoTest = new ConsoFidele(
        "Lifo", "Paul",
        "lifo.paul@gmail.com", "0600000000",
        new SimpleDateFormat("yyyy-MM-dd").parse("1961-01-03"),
        new SimpleDateFormat("yyyy-MM-dd").parse("2017-01-05")
    );

    // Le compteur de points doit être à 0 à la création
    assertEquals("L'initialisation des points doit être à 0", 0.0, consoTest.getPointsFidelite());
}
```

---

### 2.5 — Test unitaire : ajout de montants et calcul des paliers

```java
public void testAddMontant() throws Exception {
    ConsoFidele consoTest = new ConsoFidele(
        "Lifo", "Paul",
        "lifo.paul@gmail.com", "0600000000",
        new SimpleDateFormat("yyyy-MM-dd").parse("1961-01-03"),
        new SimpleDateFormat("yyyy-MM-dd").parse("2017-01-05")
    );

    // Palier 1 : 150 € (entre 100 et 200) → +10 points
    consoTest.addFidelite(3, 150);
    assertEquals("Erreur palier 100-200€", 10.0, consoTest.getPointsFidelite());

    // Palier 2 : 300 € (entre 201 et 500) → +20 points (total : 30)
    consoTest.addFidelite(3, 300);
    assertEquals("Erreur palier 201-500€", 30.0, consoTest.getPointsFidelite());

    // Palier 3 : 600 € (> 500) → +50 points (total : 80)
    consoTest.addFidelite(3, 600);
    assertEquals("Erreur palier > 500€", 80.0, consoTest.getPointsFidelite());

    // Hors palier : 50 € → +0 point (total reste à 80)
    consoTest.addFidelite(3, 50);
    assertEquals("Erreur achat inférieur à 100€", 80.0, consoTest.getPointsFidelite());
}
```

---

## Partie 3 — Programmation orientée objet

### 3.1 — Javadoc : calcul du taux de pénétration des clients fidèles

```java
/**
 * Calcule le pourcentage de ventes réalisées par des clients fidèles
 * au sein d'une liste de ventes donnée.
 *
 * @param lesVentesDuJour La collection des ventes effectuées sur une journée.
 * @return Le taux de pénétration des clients fidèles (en pourcentage)
 *         par rapport au total des ventes.
 */
```

---

### 3.2 — Méthode `getNbVentes()`

```java
public int getNbVentes() {
    return this.lesVentes.size();
}
```

---

### 3.3 — Constructeur `VenteEcommerce`

```java
public VenteEcommerce(Date uneDateVente, Conso unConso, double unMontant, String adresse, String option) {
    super(uneDateVente, unConso, unMontant); // Appel au constructeur parent Vente
    this.adresseLivraison = adresse;
    this.optionLivraison = option;
}
```

---

### 3.4 — Méthode `compareLieuVente()`

```java
public static double compareLieuVente(ArrayList<ConsoFidele> lesConsos) {
    double totalEcom = 0;
    double totalMag = 0;

    for (ConsoFidele cf : lesConsos) {
        for (Vente v : cf.getVentes()) {
            if (v instanceof VenteEcommerce) {
                totalEcom += v.getMontantVente();
            } else if (v instanceof VenteMagasin) {
                totalMag += v.getMontantVente();
            }
        }
    }

    // Protection contre la division par zéro (voir 3.5)
    if (totalEcom == 0) return 0;

    return totalMag / totalEcom;
}
```

---

### 3.5 — Gestion de la division par zéro

```java
if (totalEcom != 0) {
    return totalMag / totalEcom;
} else {
    return -1; // Valeur sentinelle indiquant l'impossibilité du calcul
}
```

---

### 3.6 — Récapitulatif de la correction (division par zéro)

```java
if (totalEcom != 0) {
    return totalMag / totalEcom;
} else {
    return -1; // Ou 0, ou lever une exception personnalisée
}
```

---

*SEKOU TANDIA — SIO2*
