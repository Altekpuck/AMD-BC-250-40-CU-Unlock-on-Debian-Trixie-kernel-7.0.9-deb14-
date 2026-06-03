# AMD BC-250 — 40 CU Unlock on Debian Trixie (kernel 7.0.9+deb14)

> **Guide communautaire** basé sur le repo original [duggasco/bc250-40cu-unlock](https://github.com/duggasco/bc250-40cu-unlock).  
> Ce tuto documente la procédure manuelle validée sur **Debian Trixie** avec le kernel `7.0.9+deb14-amd64`, où le script automatique du repo échoue en raison d'incompatibilités de structure kernel.

---

## Résultats validés

**Benchmark llama.cpp (du repo original, pp512) :**

| Config | CU actifs | tok/s (Qwen3.5 9B Q4) | Puissance | Temp |
|--------|-----------|----------------------|-----------|------|
| Stock | 24 CU | ~230 tok/s | ~95W | ~79°C |
| **40 CU débloqués** | **40 CU** | **~372 tok/s** | **~125W** | **~83°C** |
| **Gain** | **+67%** | **1.61x** | +30W | +4°C |

**Mesure réelle via Ollama + Vulkan (ce guide, Qwopus3.5-9B Q4_K_M, governor 1500 MHz/900 mV) :**

| Métrique | CPU (avant) | GPU Vulkan + 40 CU |
|----------|-------------|--------------------|
| Prompt eval | ~10 tok/s | **549 tok/s** |
| Génération | ~5 tok/s | **~49 tok/s** |
| Température (charge soutenue) | — | **61°C** |

> Débloquer les CU ne suffit pas : voir la section **Faire tourner Ollama sur GPU (Vulkan)** plus bas pour la config indispensable (Vulkan + iGPU + ROCm désactivé).

---

## Matériel et environnement

- **Carte** : AMD BC-250 (Cyan Skillfish / gfx1013 / APU PS5 reconverti)
- **PCI ID** : `1002:13fe`
- **BIOS** : P3.00
- **OS** : Debian Trixie (testing)
- **Kernel** : `7.0.9+deb14-amd64`
- **Architecture** : amd64

---

## Pourquoi le script automatique échoue

Le script `bc250-enable-40cu.sh` du repo cherche une archive `.tar.xz` des sources kernel ET attend une structure de répertoire spécifique à Debian Forky kernel 6.19. Sur Debian Trixie kernel 7.0 :

1. Le script ne trouve pas les sources kernel correctement
2. Le patch ne s'applique pas automatiquement (offset de lignes différent)
3. Le module `amdgpu.ko` compilé n'est pas trouvé au bon endroit

**Solution : application manuelle du patch pas à pas.**

---

## Prérequis

```bash
su -
apt update
apt install -y git gcc make zstd dwarves linux-headers-$(uname -r) linux-source
```

Vérifier les paquets disponibles :

```bash
apt-cache search linux-source
# Doit afficher : linux-source-7.0
```

---

## Étape 1 — Cloner le repo

```bash
git clone https://github.com/duggasco/bc250-40cu-unlock.git
cd bc250-40cu-unlock
```

---

## Étape 2 — Extraire les sources kernel

```bash
# Extraire les sources
tar -xf /usr/src/linux-source-7.0.tar.xz -C /usr/src/

# Vérifier l'extraction
ls /usr/src/
# Doit afficher : linux-source-7.0  linux-source-7.0.tar.xz
```

Créer un lien symbolique pour que le script trouve les sources :

```bash
ln -s /usr/src/linux-source-7.0 /usr/src/linux-source-7.0.9+deb14
```

---

## Étape 3 — Appliquer le patch manuellement

Le patch doit être appliqué avec `-p1` depuis la racine des sources, et non `-p5` comme tente le script automatique.

```bash
cd /usr/src/linux-source-7.0
patch -p1 < /root/bc250-40cu-unlock/patch/bc250-40cu-amdgpu.patch
```

Résultat attendu :
```
patching file drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
Hunk #2 succeeded at 10124 with fuzz 1 (offset -10 lines).
```

> **Note** : Le `fuzz 1` est normal — le kernel 7.0 a quelques lignes de décalage par rapport à la version de référence du patch (6.19). Le patch s'applique correctement malgré tout.

---

## Étape 4 — Corriger la déclaration du paramètre module

Le patch insère le code `module_param` **avant** les `#include`, ce qui cause une erreur de compilation. Il faut le déplacer **après** le dernier include.

### 4a — Vérifier le placement actuel

```bash
sed -n '25,35p' /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
```

Si vous voyez `static int bc250_cc_write_mode;` dans les includes, supprimez ces lignes (ajuster les numéros si nécessaire) :

```bash
sed -i '27,31d' /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
```

### 4b — Trouver le dernier include

```bash
grep -n "^#include" /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c | tail -3
```

Résultat sur kernel 7.0 :
```
46:#include "gfx_v10_0.h"
47:#include "gfx_v10_0_cleaner_shader.h"
48:#include "nbio_v2_3.h"
```

### 4c — Insérer le code après le dernier include (ligne 48)

```bash
sed -i '48a\
\
static int bc250_cc_write_mode;\
module_param(bc250_cc_write_mode, int, 0444);\
MODULE_PARM_DESC(bc250_cc_write_mode, "BC-250: 0=off 3=enable all");\
#define BC250_PCI_DEVICE_ID 0x13FE' /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
```

### 4d — Vérifier le résultat

```bash
sed -n '46,56p' /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c
```

Résultat attendu :
```c
#include "gfx_v10_0.h"
#include "gfx_v10_0_cleaner_shader.h"
#include "nbio_v2_3.h"

static int bc250_cc_write_mode;
module_param(bc250_cc_write_mode, int, 0444);
MODULE_PARM_DESC(bc250_cc_write_mode, "BC-250: 0=off 3=enable all");
#define BC250_PCI_DEVICE_ID 0x13FE
```

---

## Étape 5 — Corriger le header trace manquant

Le build échoue avec `fatal error: amdgpu_trace.h: No such file or directory`. Correction :

```bash
mkdir -p /usr/src/linux-headers-7.0.9+deb14-common/include/trace/../../drivers/gpu/drm/amd/amdgpu/

cp /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/amdgpu_trace.h \
   /usr/src/linux-headers-7.0.9+deb14-common/include/trace/../../drivers/gpu/drm/amd/amdgpu/
```

---

## Étape 6 — Compiler le module

```bash
cd /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/
make -C /lib/modules/$(uname -r)/build M=$(pwd) -j$(nproc) modules
```

La compilation prend **5 à 15 minutes**. Résultat attendu en fin de build :
```
  LD [M]  amdgpu.ko
```

Vérifier que le paramètre est bien présent dans le module compilé :

```bash
/sbin/modinfo /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/amdgpu.ko | grep bc250
# Attendu : parm: bc250_cc_write_mode:BC-250: 0=off 3=enable all (int)
```

---

## Étape 7 — Installer le module

```bash
# Compresser et installer
zstd /usr/src/linux-source-7.0/drivers/gpu/drm/amd/amdgpu/amdgpu.ko \
  -o /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko.zst

# Mettre à jour les dépendances modules
/sbin/depmod -a
```

---

## Étape 8 — Activer le patch et mettre à jour l'initramfs

```bash
# Créer la config modprobe
echo 'options amdgpu bc250_cc_write_mode=3' > /etc/modprobe.d/bc250-40cu.conf

# CRITIQUE : mettre à jour l'initramfs
# Sans cette étape, le système charge l'ancien module au boot
update-initramfs -u
```

---

## Étape 9 — Reboot et vérification

```bash
systemctl reboot -i
```

Après le reboot :

```bash
# Vérifier que le paramètre est actif
cat /sys/module/amdgpu/parameters/bc250_cc_write_mode
# Attendu : 3

# Vérifier les 40 CUs actifs
dmesg | grep active_cu_number
# Attendu : amdgpu 0000:01:00.0: SE 2, SH per SE 2, CU per SH 10, active_cu_number 40
```

---

## Persistance au reboot

Le patch est **permanent à chaque démarrage** grâce à trois éléments :

1. Le module `amdgpu.ko.zst` patché dans `/lib/modules/`
2. La config `/etc/modprobe.d/bc250-40cu.conf` avec `bc250_cc_write_mode=3`
3. L'initramfs mis à jour qui intègre le module patché au boot

Aucune manipulation requise après le premier reboot validé. ✅

---

## Désactiver le patch

```bash
rm /etc/modprobe.d/bc250-40cu.conf
update-initramfs -u
systemctl reboot -i
```

---

## Tester avec Ollama

```bash
apt install curl -y
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3:14b
ollama run qwen3:14b
```

Vous devriez observer environ **370 tok/s** en prefill (pp512) sur un modèle 14B Q4.

---

## Récapitulatif complet (copier-coller)

```bash
# En tant que root
su -

# Dépendances
apt update && apt install -y git gcc make zstd dwarves \
  linux-headers-$(uname -r) linux-source

# Sources kernel
tar -xf /usr/src/linux-source-7.0.tar.xz -C /usr/src/
ln -s /usr/src/linux-source-7.0 /usr/src/linux-source-7.0.9+deb14

# Cloner le repo
git clone https://github.com/duggasco/bc250-40cu-unlock.git

# Appliquer le patch
cd /usr/src/linux-source-7.0
patch -p1 < /root/bc250-40cu-unlock/patch/bc250-40cu-amdgpu.patch

# Vérifier et corriger le placement du module_param
# (voir étapes 4a-4d ci-dessus)
grep -n "^#include" drivers/gpu/drm/amd/amdgpu/gfx_v10_0.c | tail -1
# Adapter le numéro de ligne selon le résultat

# Corriger le header trace
mkdir -p /usr/src/linux-headers-7.0.9+deb14-common/include/trace/../../drivers/gpu/drm/amd/amdgpu/
cp drivers/gpu/drm/amd/amdgpu/amdgpu_trace.h \
   /usr/src/linux-headers-7.0.9+deb14-common/include/trace/../../drivers/gpu/drm/amd/amdgpu/

# Compiler
cd drivers/gpu/drm/amd/amdgpu/
make -C /lib/modules/$(uname -r)/build M=$(pwd) -j$(nproc) modules

# Installer
zstd amdgpu.ko -o /lib/modules/$(uname -r)/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko.zst
/sbin/depmod -a

# Activer et mettre à jour l'initramfs
echo 'options amdgpu bc250_cc_write_mode=3' > /etc/modprobe.d/bc250-40cu.conf
update-initramfs -u
systemctl reboot -i

# Après reboot — vérification
cat /sys/module/amdgpu/parameters/bc250_cc_write_mode  # doit afficher 3
dmesg | grep active_cu_number                           # doit afficher 40
```

---

## Faire tourner Ollama sur GPU (Vulkan) — le maillon manquant

Débloquer les 40 CU ne suffit pas : encore faut-il qu'Ollama **utilise** le GPU. Sur cette puce (gfx1013), c'est l'étape la plus piégeuse, car **ROCm ne supporte PAS gfx1013** et plante. La seule voie GPU fonctionnelle est **Vulkan (RADV/Mesa)**.

### Symptômes du problème

Sans configuration, Ollama tombe sur l'un de ces deux écueils :

```
# ROCm plante :
error: Cannot read .../rocblas/library/TensileLibrary.dat: Illegal seek for GPU arch : gfx1013

# Vulkan désactivé par défaut :
experimental Vulkan support disabled. To enable, set OLLAMA_VULKAN=1

# Et sur les versions récentes d'Ollama, l'iGPU est rejeté :
dropping integrated GPU; to enable, set OLLAMA_IGPU_ENABLE=1
```

Résultat : Ollama tourne en **CPU** (~5 tok/s).

### La solution — 3 variables d'environnement

Le BC-250 a une mémoire **unifiée** (GDDR6 partagée CPU/GPU), donc Ollama le classe comme *integrated GPU* et le rejette par défaut sur les versions récentes. Il faut **trois** variables :

```bash
systemctl edit ollama
```

Ajouter dans la section éditable :

```ini
[Service]
Environment="OLLAMA_VULKAN=1"
Environment="OLLAMA_IGPU_ENABLE=1"
Environment="ROCR_VISIBLE_DEVICES="
```

| Variable | Rôle |
|----------|------|
| `OLLAMA_VULKAN=1` | Active le backend Vulkan (la seule voie GPU sur gfx1013) |
| `OLLAMA_IGPU_ENABLE=1` | Autorise le GPU à mémoire unifiée (sinon rejeté) |
| `ROCR_VISIBLE_DEVICES=` *(vide)* | Désactive ROCm qui plante sur gfx1013 |

Puis :

```bash
systemctl daemon-reload
systemctl restart ollama
```

### Prérequis Vulkan

```bash
apt install -y mesa-vulkan-drivers vulkan-tools libvulkan1
vulkaninfo --summary | grep deviceName
# Doit afficher : AMD BC-250 (RADV GFX1013)
```

### Vérification

```bash
# Pendant une inférence :
ollama ps
# Colonne PROCESSOR doit afficher du GPU (et non 100% CPU)

# Dans les logs, on doit voir le device Vulkan :
journalctl -u ollama --no-pager | grep -i "Vulkan0"
# library=Vulkan ... description="AMD BC-250 (RADV GFX1013)"
```

### Performances mesurées (40 CU + Vulkan, governor 1500 MHz/900 mV)

Modèle de test : **Qwopus3.5-9B-v3 Q4_K_M** (Qwen3.5-9B distillé sur Claude Opus)

| Métrique | CPU (avant) | GPU Vulkan + 40 CU |
|----------|-------------|--------------------|
| Prompt eval | ~10 tok/s | **549 tok/s** |
| Génération (eval) | ~5 tok/s | **~49 tok/s** |
| Température (charge 1 min) | — | **61°C** |

> ⚠️ **Note version Ollama** : il faut une version récente d'Ollama pour les architectures modernes comme `qwen35` (Qwen3.5). Une version trop ancienne renvoie `unknown model architecture: 'qwen35'`. Mettre à jour avec `curl -fsSL https://ollama.com/install.sh | sh`, puis re-vérifier que les 3 variables d'environnement sont toujours présentes (l'installeur peut réécrire le service).

> ⚠️ **Limite mémoire Vulkan** : sur cet APU, Vulkan n'expose qu'environ **7,9 Go** sur les 16 Go unifiés. Suffisant pour un modèle 9B en Q4 (~5,6 Go) avec un contexte raisonnable, mais pas pour des modèles plus gros sans ajustement.

---

## Crédits

- **[duggasco](https://github.com/duggasco/bc250-40cu-unlock)** — recherche originale, patch kernel, documentation
- **filippor** — tests indépendants, cyan-skillfish-governor
- **Claude (Anthropic)** — analyse, découverte registre SPI, tooling
- **Communauté BC-250 Discord** — guidance thermique et tests fleet

---

## Licence

GPL-2.0 — même licence que le noyau Linux.
