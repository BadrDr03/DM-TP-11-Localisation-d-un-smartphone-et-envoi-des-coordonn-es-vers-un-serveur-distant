# DM-TP-11-Localisation-d-un-smartphone-et-envoi-des-coordonn-es-vers-un-serveur-distant

Objectifs pédagogiques
À la fin de cette séance, il devient possible de :
récupérer la latitude et la longitude d’un smartphone ;
comprendre le rôle des permissions Android liées à la localisation et au réseau ;
envoyer des données d’une application Android vers un service PHP ;
enregistrer les coordonnées dans une base MySQL ;
structurer un mini projet mobile connecté à un backend.

---

Résultat attendu
À l’issue du TP, l’application doit :
détecter une position géographique ;
afficher les informations récupérées ;
envoyer latitude, longitude, date et identifiant du téléphone au serveur ;
insérer ces données dans la table position de MySQL.

---

Vue d’ensemble de l’architecture
Le système comporte deux parties.
Partie serveur
Elle contient :
une base de données MySQL ;
des classes PHP ;
un script recevant les données envoyées par le smartphone.
Partie mobile
Elle contient :
les permissions Android ;
la récupération des coordonnées GPS ;
l’envoi HTTP avec Volley.
Le fonctionnement général est le suivant :
le smartphone obtient une nouvelle position ;
l’application prépare une requête HTTP POST ;
le serveur PHP reçoit les paramètres ;
le serveur crée un objet métier Position ;
les données sont enregistrées dans la base

---

Partie serveur — Base de données MySQL
Étape 1 — Créer la base de données
Créer une base nommée localisation.
Étape 2 — Créer la table position
Exécuter le script SQL suivant :
CREATE DATABASE IF NOT EXISTS localisation
CHARACTER SET utf8mb4
COLLATE utf8mb4_unicode_ci;

USE localisation;

CREATE TABLE position (
    id INT AUTO_INCREMENT PRIMARY KEY,
    latitude DOUBLE NOT NULL,
    longitude DOUBLE NOT NULL,
    date_position DATETIME NOT NULL,
    imei VARCHAR(50) NOT NULL
);
Explication
Cette table stocke chaque position envoyée depuis l’application Android.
id identifie chaque enregistrement ;
latitude contient la coordonnée nord-sud ;
longitude contient la coordonnée est-ouest ;
date_position stocke la date et l’heure d’envoi ;
imei représente l’identifiant du smartphone utilisé dans le cadre du TP. La structure source du sujet prévoit déjà les champs latitude, longitude, date et imei. Ici, seul le nom date_position est adopté pour une meilleure lisibilité.
Vérification
Ouvrir phpMyAdmin et vérifier que la table position a bien été créée.


---


Partie serveur — Développement PHP
Étape 3 — Organiser l’arborescence du projet
Créer une structure de projet comme suit :
localisation/
│
├── classe/
│   └── Position.php
│
├── connexion/
│   └── Connexion.php
│
├── dao/
│   └── IDao.php
│
├── service/
│   └── PositionService.php
│
└── createPosition.php
Explication
Cette organisation permet de séparer :
les classes métier ;
la gestion de la connexion ;
le contrat DAO ;
le service d’accès aux données ;
le point d’entrée HTTP.


Étape 4 — Créer la classe métier Position
Créer le fichier classe/Position.php :
<?php
class Position {
    private $id;
    private $latitude;
    private $longitude;
    private $datePosition;
    private $imei;

    public function __construct($id, $latitude, $longitude, $datePosition, $imei) {
        $this->id = $id;
        $this->latitude = $latitude;
        $this->longitude = $longitude;
        $this->datePosition = $datePosition;
        $this->imei = $imei;
    }

    public function getId() {
        return $this->id;
    }

    public function getLatitude() {
        return $this->latitude;
    }

    public function getLongitude() {
        return $this->longitude;
    }

    public function getDatePosition() {
        return $this->datePosition;
    }

    public function getImei() {
        return $this->imei;
    }

    public function setId($id) {
        $this->id = $id;
    }

    public function setLatitude($latitude) {
        $this->latitude = $latitude;
    }

    public function setLongitude($longitude) {
        $this->longitude = $longitude;
    }

    public function setDatePosition($datePosition) {
        $this->datePosition = $datePosition;
    }

    public function setImei($imei) {
        $this->imei = $imei;
    }
}
?>
Explication du code
Cette classe représente l’objet manipulé côté serveur.
Chaque objet Position correspond à une ligne de la table position.
Les attributs sont privés afin d’encapsuler les données. L’accès aux valeurs se fait à travers des getters et des setters. Cette approche permet de mieux structurer le code et de préparer les étudiants à une conception orientée objet plus rigoureuse. La structure du sujet prévoit déjà une classe Position avec id, latitude, longitude, date et imei.


Étape 5 — Créer la classe Connexion
Créer le fichier connexion/Connexion.php :
<?php
class Connexion {
    private $connexion;

    public function __construct() {
        $host = 'localhost';
        $dbname = 'localisation';
        $login = 'root';
        $password = '';

        try {
            $this->connexion = new PDO(
                "mysql:host=$host;dbname=$dbname;charset=utf8mb4",
                $login,
                $password
            );
            $this->connexion->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        } catch (Exception $e) {
            die('Erreur de connexion : ' . $e->getMessage());
        }
    }

    public function getConnexion() {
        return $this->connexion;
    }
}
?>
Explication du code
Cette classe centralise la connexion à la base de données.
Elle a deux rôles :
ouvrir une connexion PDO ;
fournir cette connexion aux autres classes.
L’usage de PDO facilite le travail avec MySQL et permet de préparer des requêtes SQL plus propres. La configuration du sujet s’appuie déjà sur PDO et une base localisation. Ici, la connexion est simplement rendue plus lisible et plus robuste.


Étape 6 — Créer l’interface IDao
Créer le fichier dao/IDao.php :
<?php
interface IDao {
    public function create($obj);
    public function update($obj);
    public function delete($obj);
    public function getById($obj);
    public function getAll();
}
?>
Explication du code
Cette interface définit les opérations classiques d’accès aux données :
insertion ;
mise à jour ;
suppression ;
recherche ;
récupération de tous les enregistrements.
Même si toutes les méthodes ne sont pas encore utilisées, cette interface introduit une organisation professionnelle du code.


Étape 7 — Créer le service PositionService
Créer le fichier service/PositionService.php :
<?php
include_once 'dao/IDao.php';
include_once 'classe/Position.php';
include_once 'connexion/Connexion.php';

class PositionService implements IDao {
    private $connexion;

    public function __construct() {
        $this->connexion = new Connexion();
    }

    public function create($position) {
        $sql = "INSERT INTO position(latitude, longitude, date_position, imei)
                VALUES(:latitude, :longitude, :date_position, :imei)";
        $stmt = $this->connexion->getConnexion()->prepare($sql);
        $stmt->execute([
            ':latitude' => $position->getLatitude(),
            ':longitude' => $position->getLongitude(),
            ':date_position' => $position->getDatePosition(),
            ':imei' => $position->getImei()
        ]);
    }

    public function update($obj) {
    }

    public function delete($obj) {
    }

    public function getById($obj) {
    }

    public function getAll() {
    }
}
?>
Explication du code
Cette classe joue le rôle de service d’accès aux données.
La méthode create() :
prépare une requête SQL ;
remplace les paramètres par les valeurs de l’objet Position ;
exécute l’insertion dans la table.
L’utilisation d’une requête préparée améliore la qualité du code et rend la requête plus claire. Le sujet prévoit déjà une méthode create() dans PositionService pour insérer latitude, longitude, date et imei.


Étape 8 — Créer le script createPosition.php
Créer le fichier createPosition.php à la racine du projet :
<?php
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    include_once 'service/PositionService.php';
    create();
}

function create() {
    $latitude = $_POST['latitude'];
    $longitude = $_POST['longitude'];
    $datePosition = $_POST['date_position'];
    $imei = $_POST['imei'];

    $service = new PositionService();
    $position = new Position(null, $latitude, $longitude, $datePosition, $imei);
    $service->create($position);

    echo "Position enregistrée avec succès";
}
?>
Explication du code
Ce script constitue le point d’entrée côté serveur.
Lorsque l’application Android envoie une requête POST :
les paramètres sont récupérés avec $_POST ;
un objet Position est créé ;
le service PositionService insère la position en base ;
un message de confirmation est renvoyé.
Le sujet indique aussi que l’adresse IP du client peut être obtenue avec $_SERVER['REMOTE_ADDR'], ce qui peut être ajouté ultérieurement comme amélioration.
Test rapide
Ouvrir Postman ou utiliser un formulaire HTML pour envoyer :
latitude
longitude
date_position
imei
Puis vérifier l’insertion dans MySQL.
