---
icon: lucide/app-window
---

# Mission 1: Demande de congé C&#35;

## Présentation

### Objectif

Pour réussir la mission l'application doit présenter ceci:

- Une page de connexion
- Une page utilisateur pour pouvoir consulter son solde de congé, faire des demandes de congé et savoir si sa demande est accepter
- Une page RH qui permet d'accepter ou refuser les différents congé présent en attente.

## Base de donnée

Ci-joint la photo du MCD

![MCD](image/mission1/mcdmission1.JPG)

On peut ajouter à ceci l'ajout de procedure stocké et des événements.

L'événements sert à remettre des nouveaux congé tout les X du mois et l'autre à faire chaque an le reset du soldes de congé.

Les procédures stocké sont appelé par les événements pour déclancher faire les modification.

### Contenu des procédures stocké

```sql title='Ajout Congé Mensuels'
BEGIN
    UPDATE praticien
    SET praticien.Solde_congé = praticien.Solde_congé + 0.63;
END
```

```sql title='Remise Congé Annuelle'
BEGIN
    UPDATE praticien
    SET praticien.Ancien_Solde_Congé = praticien.Solde_congé,
        praticien.Solde_congé = 0; -
END
```

## C&#35;

### Composition des pages

#### Connexion

![Connexion](image/mission1/Cconnexion.JPG)

Nous remarquons les deux champs possible, **adresse mail** et **mot de passe** qui existe pour chaque praticiens de la base de données.

#### Interface Utilisateur

![InterfaceUser](image/mission1/interfaceuserC.JPG)

Il y a deux champs important qui sont la **Date de début** et la **date de fin** pour demander les congé on remarque que en bas a droite on a le solde de congé actuellement.

#### Interface RH

![InterfaceRH](image/mission1/interfaceRHC.JPG)

On remarque une ListeView ou iront tout les praticiens qui demande des congé. Pour accepter il suffit de séléctionner et cliquer soit sur accepté ou sois refusé.

### Les parties complexe du codes


#### Interface Admin

Le code que l'on remarque ci-contre sert à récupéré toute les demandes et les affichés. 

```c# title='Admin_Load'
bdd = new BDD("AP", "APSIO2", "172.23.48.2", "gsb");
bdd.Connecter();
listeConge = bdd.ChargerConge(); //récupéré les congé
lvadmin.View = View.Details;
lvadmin.Columns.Add("ID",100);
lvadmin.Columns.Add("Nom",120);
lvadmin.Columns.Add("Prénom",120);
lvadmin.Columns.Add("Date Début", 110);
lvadmin.Columns.Add("Date Fin", 110);

foreach (Conge con in listeConge) //les affiché dans une listes views
{
    if (con.etat == "1")
    {
        var item = new ListViewItem(con.id.ToString());
        item.SubItems.Add(con.praticien.nom);
        item.SubItems.Add(con.praticien.prenom);
        item.SubItems.Add(con.date_debut.ToString("yyyy-MM-dd"));
        item.SubItems.Add(con.date_fin.ToString("yyyy-MM-dd"));
        lvadmin.Items.Add(item);
    }
}
```

#### BDD

Fonctionnement du chargé congé:

```c#
    List<Conge> listeConge = new List<Conge>();
    MySqlCommand cmd = null;
    MySqlDataReader reader = null;

    try
    {
        // ✅ Vérifie et ouvre la connexion si nécessaire
        if (connection.State != ConnectionState.Open)
            connection.Open();

        string requete = @"
    SELECT c.id, c.id_praticien, c.date_debut, c.date_fin, c.état,
           p.nom, p.prenom, p.id_ville, p.solde_congé
    FROM congé c
    INNER JOIN praticien p ON c.id_praticien = p.id
    WHERE c.état = 1;";

        cmd = new MySqlCommand(requete, connection);
        reader = cmd.ExecuteReader();

        while (reader.Read())
        {
            int id = Convert.ToInt32(reader["id"]);
            int id_praticien = Convert.ToInt32(reader["id_praticien"]);
            DateTime date_debut = Convert.ToDateTime(reader["date_debut"]);
            DateTime date_fin = Convert.ToDateTime(reader["date_fin"]);
            string etat = reader["état"].ToString();

            string nom = reader["nom"].ToString();
            string prenom = reader["prenom"].ToString();
            int id_ville = Convert.ToInt32(reader["id_ville"]);
            decimal solde_conge = Convert.ToDecimal(reader["solde_congé"]);
            //création des objets
            Practicien prat = new Practicien(id_praticien, nom, prenom, id_ville, solde_conge); //création des praticiens
            Conge conge = new Conge(id, id_praticien, date_debut, date_fin, etat); //création des conges
            conge.praticien = prat;

            listeConge.Add(conge);
        }
    }
    catch (Exception ex)
    {
        MessageBox.Show("Erreur SQL : " + ex.Message);
    }
    finally
    {
        if (reader != null && !reader.IsClosed)
            reader.Close();

        if (cmd != null)
            cmd.Dispose();

    }

    return listeConge;

```

Affiché les messages lors de la connexion.

```c#
try
{
    string requete = "SELECT id_notif, message FROM notification WHERE id_receveur = @id AND id_etat = 2";
     //récupére que les notif du prat connecté et seulement les pas lu
    MySqlCommand cmd = new MySqlCommand(requete, connection);
    cmd.Parameters.AddWithValue("@id", idp);

    using (MySqlDataReader reader = cmd.ExecuteReader())
    {
        while (reader.Read())
        {
            string message = reader["message"].ToString();
            int idNotification = Convert.ToInt32(reader["id_notif"]);

            if (!string.IsNullOrEmpty(message))
            {
                DialogResult result = MessageBox.Show(
                    message,
                    "Message du RH",
                    MessageBoxButtons.OK,
                    MessageBoxIcon.Information
                );

                if (result == DialogResult.OK)
                {
                    reader.Close();
                    MarquerCommeLu(idNotification);

                    AfficherMessage(idp);
                    break;
                }
            }
        }
    }
}
catch (Exception ex)
{
    MessageBox.Show("Erreur : " + ex.Message);
}
```



