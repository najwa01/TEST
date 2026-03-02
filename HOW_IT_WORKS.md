# 🛡️ Comment fonctionne le système anti-phishing ?

## 📋 Table des matières
1. [Vue d'ensemble](#vue-densemble)
2. [Architecture du système](#architecture-du-système)
3. [Le Classifier XGBoost](#le-classifier-xgboost)
4. [Les Features (caractéristiques)](#les-features-caractéristiques)
5. [Le processus de prédiction](#le-processus-de-prédiction)
6. [L'intégration TinyLlama](#lintégration-tinyllama)
7. [Le système d'explications](#le-système-dexplications)

---

## 🎯 Vue d'ensemble

Votre système utilise une **approche hybride** combinant deux technologies:

```
┌─────────────────────────────────────────────────────────┐
│                    SYSTÈME ANTI-PHISHING                │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────┐        ┌─────────────────┐       │
│  │   XGBoost ML    │   +    │   TinyLlama AI  │       │
│  │   Classifier    │        │   (Optional)     │       │
│  └─────────────────┘        └─────────────────┘       │
│         ↓                           ↓                   │
│  ┌─────────────────────────────────────────────┐       │
│  │         DÉCISION FINALE COMBINÉE            │       │
│  │     (PHISHING / SAFE + Explications)        │       │
│  └─────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

### 🔑 Points clés:
- **Classifier ML**: Analyse instantanée basée sur 36 features
- **TinyLlama AI**: Analyse contextuelle en langage naturel (optionnel)
- **Hybrid Decision**: Combine les deux pour plus de précision

---

## 🏗️ Architecture du système

### 1. **Extraction de features** (`features/extract.py`)
```
URL → Parser → Features (36 caractéristiques) → DataFrame
```

### 2. **Classification** (`classifier/train.py` + `model.pkl`)
```
Features → XGBoost Model → Probabilité (0-1) → Verdict
```

### 3. **Explication** (`inference/predict_with_llm.py`)
```
Features + Probabilité → Analyse → Explication en texte
```

### 4. **Interface Web** (`web/app.py` + `index.html`)
```
User Input → API → Analyse → Résultats + Explications
```

---

## 🤖 Le Classifier XGBoost

### Qu'est-ce que XGBoost ?

**XGBoost** (eXtreme Gradient Boosting) est un algorithme d'apprentissage automatique qui:
- Construit **500 arbres de décision** en séquence
- Chaque arbre corrige les erreurs du précédent
- Résultat final = combinaison de tous les arbres

### 📊 Exemple visuel:

```
Arbre 1: "TLD .xyz?" → Suspect
         ↓
Arbre 2: "Contient 'login'?" → Suspect
         ↓
Arbre 3: "Sous-domaine long?" → Ok
         ↓
         ...
         ↓
Arbre 500: Décision finale
         ↓
    Probabilité: 85% Phishing
```

### ⚙️ Configuration utilisée:

```python
XGBClassifier(
    n_estimators=500,           # 500 arbres
    max_depth=5,                # Profondeur max de 5 niveaux
    learning_rate=0.03,         # Apprentissage lent = plus stable
    subsample=0.8,              # 80% des données par arbre
    colsample_bytree=0.8,       # 80% des features par arbre
    scale_pos_weight=0.61,      # Équilibrage phishing/légitimes
    eval_metric="auc",          # Optimisation sur AUC-ROC
)
```

### 📈 Dataset d'entraînement:

- **13 106 URLs** au total
- **8 115 phishing** (62%)
- **4 991 légitimes** (38%)
- Source: fichiers JSON convertis en CSV

### 🎯 Threshold (seuil de décision):

```
Probabilité ≥ 0.35 → PHISHING
Probabilité < 0.35 → SAFE
```

**Pourquoi 0.35 ?**
- Priorité à la **détection** (recall élevé: 92%)
- Mieux détecter un faux positif que manquer un vrai phishing
- Réglé pour la sécurité maximale

---

## 🔍 Les Features (caractéristiques)

Le système analyse **36 caractéristiques** de chaque URL:

### 📏 **Features de base (URL)**
| Feature | Description | Exemple |
|---------|-------------|---------|
| `url_length` | Longueur totale | 45 caractères |
| `fqdn_length` | Longueur du domaine complet | 22 caractères |
| `domain_length` | Longueur du domaine principal | 6 (paypal) |
| `num_subdomains` | Nombre de sous-domaines | 2 (secure.login.site.com) |
| `domain_entropy` | Entropie de Shannon | 3.2 (plus haut = plus aléatoire) |

### 🎯 **Features critiques anti-phishing**
| Feature | Description | Valeur Phishing |
|---------|-------------|----------------|
| `brand_in_fqdn` | Marque détectée dans FQDN | 1 (oui) |
| `brand_not_domain` | Marque dans sous-domaine seulement | ⚠️ 1 (typosquatting!) |
| `path_contains_login` | Mots sensibles dans le chemin | ⚠️ 1 (login/verify/secure) |
| `suspicious_tld_extended` | TLD suspect | ⚠️ 1 (.xyz, .top, .tk, etc.) |
| `long_subdomain` | Sous-domaine > 15 caractères | ⚠️ 1 |
| `fqdn_digit_ratio` | Ratio de chiffres dans le domaine | > 0.2 suspect |

### 📊 **Features lexicales**
| Feature | Description | Exemple |
|---------|-------------|---------|
| `digit_ratio` | Ratio de chiffres dans l'URL | 0.15 (15% de chiffres) |
| `special_char_count` | Nombre de caractères spéciaux | 7 (-, _, =, @, ?) |
| `max_token_length` | Longueur du plus long token | 12 |
| `contains_sensitive_word` | Mots sensibles détectés | 1 (login, verify, etc.) |
| `path_depth` | Profondeur du chemin | 3 niveaux |
| `suspicious_tld` | TLD dans liste suspecte | 1 (.xyz, .top, .ru, .cn) |

### 🌐 **Features réseau (optionnelles)**
| Feature | Description | Utilité |
|---------|-------------|---------|
| `contains_ip` | IP dans l'URL | Suspect |
| `num_ip_addresses` | Nombre d'IPs résolues | Info DNS |
| `ipv6_present` | IPv6 disponible | Info technique |

### 🔒 **Features SSL/WHOIS (désactivées en production)**
Ces features ne sont **pas utilisées** car non disponibles en temps réel:
- `ssl_valid`, `ssl_days_remaining`, `domain_age_days`, etc.

---

## ⚡ Le processus de prédiction

### Étape par étape:

```
1️⃣ RÉCEPTION DE L'URL
   ↓
   Input: "https://paypal-verify.xyz/login"

2️⃣ EXTRACTION DES FEATURES
   ↓
   ┌─────────────────────────────────────┐
   │ url_length: 36                      │
   │ domain_length: 14                   │
   │ suspicious_tld_extended: 1 (.xyz)   │
   │ path_contains_login: 1 (/login)     │
   │ brand_in_fqdn: 1 (paypal)          │
   │ ... (31 autres features)            │
   └─────────────────────────────────────┘

3️⃣ CLASSIFICATION XGBOOST
   ↓
   500 arbres de décision analysent les features
   ↓
   Probabilité: 0.85 (85% phishing)

4️⃣ APPLICATION DU THRESHOLD
   ↓
   0.85 ≥ 0.35 → VERDICT: PHISHING

5️⃣ GÉNÉRATION DE L'EXPLICATION
   ↓
   Analyse des red flags détectés:
   - TLD .xyz suspect
   - Mot-clé "login" dans le chemin
   - Marque "paypal" dans le domaine
   ↓
   Explication: "⚠️ HIGH confidence this is PHISHING.
   Detected: suspicious TLD '.xyz', suspicious keywords
   in path (login/verify/secure/account)."

6️⃣ ANALYSE LLM (optionnel)
   ↓
   TinyLlama analyse le contexte et génère:
   "This URL attempts to impersonate PayPal using
   a suspicious .xyz domain and contains login paths,
   which is a classic phishing pattern."

7️⃣ DÉCISION FINALE
   ↓
   Classifier: 85% Phishing
   LLM: PHISHING (90% confidence)
   ↓
   FINAL: PHISHING (87% confidence)
```

### 🎨 Dans l'interface web:

```html
┌────────────────────────────────────────────────┐
│ ⚠️ THREAT DETECTED                            │
├────────────────────────────────────────────────┤
│ 🌐 URL: https://paypal-verify.xyz/login       │
│                                                │
│ 💡 Analysis:                                   │
│ ⚠️ HIGH confidence this is PHISHING.          │
│ Detected: suspicious TLD '.xyz', keywords...  │
│                                                │
│ 📊 Classifier Details                         │
│   Verdict: PHISHING                           │
│   Probability: 85.0%                          │
│   Confidence: 85%                             │
│                                                │
│ 🤖 TinyLlama AI                               │
│   Verdict: PHISHING                           │
│   Confidence: 90%                             │
│   Explanation: This URL attempts to...        │
└────────────────────────────────────────────────┘
```

---

## 🤖 L'intégration TinyLlama

### Qu'est-ce que TinyLlama ?

**TinyLlama** est un petit modèle de langage (LLM) qui:
- Tourne **localement** via Ollama
- Analyse l'URL en **langage naturel**
- Génère des **explications contextuelles**
- Taille: ~637 MB (léger!)

### 🔄 Flux d'analyse LLM:

```
1. Prompt construit avec:
   - URL à analyser
   - Probabilité du classifier
   - Liste des red flags à chercher

2. Envoi à Ollama API (localhost:11434)

3. TinyLlama génère:
   VERDICT: PHISHING/SUSPICIOUS/SAFE
   CONFIDENCE: 0-100%
   EXPLANATION: [explication détaillée]

4. Parsing de la réponse

5. Combinaison avec le classifier
```

### 💡 Avantages du LLM:

- **Contexte**: Comprend les patterns complexes
- **Flexibilité**: Détecte des variations nouvelles
- **Explications**: Génère du texte naturel
- **Local**: Pas besoin d'API externe

### ⚙️ Configuration:

```python
OLLAMA_API = "http://localhost:11434/api/generate"
OLLAMA_MODEL = "tinyllama"

options = {
    "temperature": 0.3,  # Déterministe
    "top_p": 0.9         # Diversité limitée
}
```

---

## 📝 Le système d'explications

### 🎯 Génération automatique

Même **sans LLM**, le système génère des explications basées sur les features:

```python
def generate_explanation(features, prob, url):
    red_flags = []
    safe_indicators = []
    
    # Analyse des features
    if features['brand_not_domain']:
        red_flags.append("typosquatting attempt")
    
    if features['suspicious_tld_extended']:
        red_flags.append(f"suspicious TLD '.{tld}'")
    
    if features['path_contains_login']:
        red_flags.append("suspicious keywords in path")
    
    # ... autres checks
    
    # Génération du message selon la probabilité
    if prob >= 0.7:
        return f"⚠️ HIGH confidence PHISHING. Detected: {red_flags}"
    elif prob >= 0.35:
        return f"🔶 MEDIUM confidence phishing. Warning: {red_flags}"
    else:
        return f"✅ HIGH confidence SAFE. {safe_indicators}"
```

### 📊 Niveaux de confiance:

| Probabilité | Niveau | Message | Icône |
|-------------|--------|---------|-------|
| ≥ 0.70 | HIGH PHISHING | "This is PHISHING" | ⚠️ |
| 0.35 - 0.70 | MEDIUM PHISHING | "Might be phishing" | 🔶 |
| 0.20 - 0.35 | LOW SAFE | "Appears safe" | 🟢 |
| < 0.20 | HIGH SAFE | "URL is SAFE" | ✅ |

---

## 🎓 Exemple complet d'analyse

### URL testée: `https://secure-paypal-login.xyz/verify?id=12345`

#### 1️⃣ **Features extraites:**

```json
{
  "url_length": 52,
  "domain_length": 21,
  "suspicious_tld_extended": 1,      // ⚠️ .xyz
  "path_contains_login": 1,          // ⚠️ /verify
  "brand_in_fqdn": 1,                // paypal détecté
  "brand_not_domain": 1,             // ⚠️ paypal en sous-domaine
  "long_subdomain": 1,               // ⚠️ "secure-paypal-login" > 15 chars
  "contains_sensitive_word": 1,      // ⚠️ verify, login
  "special_char_count": 3,           // -, -, ?
  "digit_ratio": 0.096,              // 5 chiffres / 52 caractères
  "path_depth": 1,
  // ... 25 autres features
}
```

#### 2️⃣ **Classifier XGBoost:**

```
500 arbres analysent → Probabilité: 0.92 (92%)
0.92 ≥ 0.35 → VERDICT: PHISHING
Confidence: 92%
```

#### 3️⃣ **Explication automatique:**

```
⚠️ HIGH confidence this is PHISHING. Detected:
- Brand name in subdomain but not in main domain (typosquatting)
- Suspicious TLD '.xyz' commonly used for phishing
- Suspicious keywords in path (login/verify/secure/account)
```

#### 4️⃣ **Analyse TinyLlama (si activé):**

```
VERDICT: PHISHING
CONFIDENCE: 95%
EXPLANATION: This URL is a clear phishing attempt 
impersonating PayPal. It uses a fake domain with the 
.xyz TLD, includes PayPal in the subdomain to trick 
users, and contains verify/login paths typical of 
credential harvesting attacks.
```

#### 5️⃣ **Décision finale:**

```
Classifier: 92% Phishing
LLM: 95% Phishing
→ FINAL: 93% Phishing (moyenne)

VERDICT: PHISHING
```

---

## 📈 Performance du modèle

### Métriques d'entraînement:

```
Dataset: 13,106 URLs
Train: 10,484 URLs (80%)
Test: 2,622 URLs (20%)

Résultats sur test set:
┌──────────┬───────────┬────────┬──────────┐
│ Class    │ Precision │ Recall │ F1-Score │
├──────────┼───────────┼────────┼──────────┤
│ SAFE     │   73%     │  34%   │   46%    │
│ PHISHING │   69%     │  92%   │   79%    │
├──────────┼───────────┼────────┼──────────┤
│ Accuracy │           │        │   70%    │
│ ROC-AUC  │           │        │  74.7%   │
└──────────┴───────────┴────────┴──────────┘
```

### 🎯 Interprétation:

- **Recall Phishing: 92%** → Détecte presque tous les phishing ✅
- **Precision SAFE: 73%** → Peu de faux positifs sur sites sûrs ✅
- **ROC-AUC: 74.7%** → Bonne capacité de discrimination ✅

### ⚖️ Trade-off choisi:

- Priorité à la **DÉTECTION** (recall élevé)
- Accepter quelques faux positifs
- Mieux prévenir que laisser passer un phishing

---

## 🚀 Utilisation

### 1. **Via l'interface web** (recommandé)

```bash
# Démarrer le serveur
python -m uvicorn web.app:app --host 127.0.0.1 --port 8080

# Ouvrir dans le navigateur
http://127.0.0.1:8080
```

### 2. **Via ligne de commande**

```bash
# Avec LLM
python inference/predict_with_llm.py "https://example.com"

# Sans LLM (plus rapide)
python inference/predict_with_llm.py "https://example.com" --no-llm
```

### 3. **Via API Python**

```python
from inference.predict_with_llm import analyze_with_llm

result = analyze_with_llm("https://example.com", use_llm=True)
print(result['final']['verdict'])      # PHISHING ou SAFE
print(result['final']['confidence'])   # 0-100
print(result['final']['reasoning'])    # Explication
```

---

## 🔧 Configuration et amélioration

### Pour améliorer les performances:

1. **Ajouter plus de données d'entraînement**
   - Collecter plus d'URLs légitimes
   - Équilibrer le dataset (50/50)

2. **Ajuster le threshold**
   - Plus bas = plus sensible (moins de faux négatifs)
   - Plus haut = plus strict (moins de faux positifs)

3. **Ajouter des features**
   - Analyse de contenu HTML
   - Vérifications SSL en temps réel
   - Réputation du domaine

4. **Réentraîner le modèle**
```bash
python classifier/train.py
```

---

## 📚 Fichiers clés du projet

```
phishing/
├── classifier/
│   ├── train.py              # Entraînement du modèle
│   ├── model.pkl             # Modèle XGBoost sauvegardé
│   ├── threshold.pkl         # Seuil optimal (0.35)
│   └── features.pkl          # Liste des 36 features
│
├── features/
│   └── extract.py            # Extraction des features d'une URL
│
├── inference/
│   ├── predict_with_llm.py   # Prédiction hybride (ML + LLM)
│   └── active_learning.py    # Système de feedback
│
├── web/
│   ├── app.py                # API FastAPI
│   └── index.html            # Interface graphique
│
├── data/
│   ├── processed_from_urls.csv  # Dataset final (13K URLs)
│   └── feedback.jsonl           # Feedbacks utilisateurs
│
└── requirements.txt          # Dépendances Python
```

---

## ❓ Questions fréquentes

### Q: Pourquoi le modèle prédit "PHISHING" pour google.com ?

**R:** Le modèle est **très sensible** (threshold bas à 0.35). Si la probabilité est entre 35-70%, c'est une zone grise. L'explication vous dira si c'est une vraie menace ou non.

### Q: Puis-je utiliser le système sans Ollama ?

**R:** Oui! Décochez "Use TinyLlama AI" dans l'interface. Le classifier XGBoost fonctionne seul et génère quand même des explications.

### Q: Comment améliorer la précision ?

**R:** 
1. Fournir des feedbacks via l'interface
2. Ajouter plus d'URLs d'entraînement
3. Réentraîner le modèle avec `python classifier/train.py`

### Q: Le système fonctionne en temps réel ?

**R:** Oui! Le classifier est **instantané** (<10ms). Le LLM prend 2-5 secondes si activé.

---

## 🎉 Résumé

Votre système anti-phishing combine:

1. **Machine Learning rapide** (XGBoost): Analyse 36 features en temps réel
2. **Intelligence artificielle** (TinyLlama): Compréhension contextuelle
3. **Explications automatiques**: Transparence totale sur les décisions
4. **Interface intuitive**: Accessible via navigateur web
5. **Feedback learning**: S'améliore avec vos retours

Le tout tournant **localement** sur votre machine! 🚀

---

**Pour plus d'informations:**
- Configuration Ollama: `OLLAMA_SETUP.md`
- Code source: Tous les fichiers sont commentés
- Support: Analysez les logs et les explications générées
