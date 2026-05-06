# Distance Map - EHPAD Agir Castres

Cartographie interactive Vue.js pour visualiser les distances entre l'EHPAD Agir Castres et le domicile de chaque employé.

## 🚀 Démarrage rapide

### Installation

```bash
cd distance-map
npm install
```

### Développement

```bash
npm run dev
```

L'application ouvrira automatiquement à `http://localhost:5173`

### Build pour production

```bash
npm run build
npm run preview
```

## 📋 Fonctionnalités

✅ **Cartographie interactive** avec OpenStreetMap  
✅ **Marqueurs** pour l'EHPAD et chaque employé  
✅ **Recherche** d'employés par nom, adresse ou ville  
✅ **Statistiques** : distance moyenne, min/max  
✅ **Popups informatifs** avec détails de chaque employé  
✅ **Lien Google Maps** pour vérifier chaque trajet  
✅ **Responsive design** avec sidebar filtrable  

## 📊 Données

Le fichier `public/liste salariés avec distances.csv` contient :
- Nom de l'employé
- Adresse complète
- Code postal et ville
- Distance en km depuis l'EHPAD
- Lien de vérification Google Maps

## 🗺️ Utilisation

1. **Consulter la carte** : visualisez l'EHPAD au centre et tous les employés autour
2. **Chercher un employé** : utilisez la barre de recherche dans la sidebar
3. **Cliquer sur un employé** : zoom automatique et affichage des informations
4. **Vérifier les distances** : cliquez sur le lien Google Maps du popup pour confirmer

## 🛠️ Technologies

- **Vue 3** - Framework JavaScript
- **Vite** - Build tool
- **Leaflet** - Cartographie
- **OpenStreetMap** - Tiles et géocodage
- **PapaParse** - Parsing CSV

## 📦 Structure du projet

```
distance-map/
├── src/
│   ├── App.vue          # Composant principal
│   ├── main.js          # Point d'entrée
│   └── style.css        # Styles globaux
├── public/
│   └── liste salariés avec distances.csv
├── index.html
├── vite.config.js
├── package.json
└── README.md
```

## 📞 Support

Pour toute question ou amélioration, consultez la documentation de :
- [Vue 3](https://vuejs.org/)
- [Leaflet](https://leafletjs.com/)
- [Vite](https://vitejs.dev/)
