# Camera_Thermique_RaspberryPi

Dans le projet,on utilise OpenCV pour détecter le visage et la température du visage humain. Notre problème sera de reconnaître les personnes qui entrent et sortent et d'enregistrer leur température, et d'avertir si cette personne a de la fièvre. Cela aidera à prévenir la propagation du COVID-19.

## I) Hardwares 
- Raspberry Pi 4 
- Une caméra pour recevoir des images, reconnaître des visages. Ici, j'utilise la webcam Logitech
- Une caméra thermique, ici j'utilise la caméra thermique MLX90640 
- Un écran avec entrée HDMI ou VGA 

## II) Installer le système d'exploitation pour Pi

Tout à bord , on télécharge le logiciel **Raspberry Pi Imager**: https://www.raspberrypi.com/software
<img width="679" alt="Screen Shot 2020-08-22 at 21 59 33" src="https://user-images.githubusercontent.com/46745468/151544621-795cf099-3bef-4f93-ae7d-5b2c8f70a32b.png">

Ensuite, j'installe les packages de **la bibliothèque** pour pouvoir accéder à la caméra thermique :

```
pip3 install matplotlib scipy numpy
sudo apt-get install -y python-smbus
sudo apt-get install -y i2c-tools
pip3 install RPI.GPIO adafruit-blinka
pip3 install adafruit-circuitpython-mlx90640
```
C'est fini, l'environnement, l'OS, à côté de la configuration supplémentaire pour le Pi.
Je vais dans le menu Démarrer, sélectionne Préférences > Configuration Raspberry Pi

![a](https://user-images.githubusercontent.com/46745468/151545459-ef1dcda5-45aa-402f-8591-c08e86aadcb5.png)

Et puis active **I2C** sur Enabled comme indiqué :

![b](https://user-images.githubusercontent.com/46745468/151545691-b3e23dc3-a3c3-49fb-b270-ffc514743b0e.png)

Après avoir effectué toutes les étapes ci-dessus, redémarre le Pi une fois pour qu'il reçoive les paramètres !

## III) Schéma de circuit la caméra de température sur le Pi

![Untitled](https://user-images.githubusercontent.com/46745468/151546243-9dd429c4-244e-4c21-aa7f-f5fd3785c689.png)

Une fois branché, J'ai fait la commande :

```
sudo i2cdetect -y 1
```
Le résultat renvoyé, c'est fait la configuration!

![é](https://user-images.githubusercontent.com/46745468/151546937-80c47da9-33a4-4416-9c0b-1fcdcb9fbbaa.png)


## IV) L'idée de l'algorithme

Le Pi lira les images de 2 caméras (1 caméra Face normale et 1 caméra thermique).

Le flux de traitement ressemblera à ceci :

- Étape 1 : Lise l'image de la caméra normale, détecte le visage dans l'image. Par exemple, il y a une face à la position rectangulaire (x1,y1)-(x2,y2).

- Étape 2 : Je procéde à la lecture de l'image de la caméra thermique (lecture en même temps) puis trouve la position (x1,y1) - (x2,y2) sur la caméra thermique et calcule les valeurs : température maximale, température minimale et température moyenne (par exemple).

- Étape 3 : Enregistre le visage et la température mesurés dans la base de données et avertissez l'orateur si la température dépasse le seuil spécifié (par exemple, 37,5).

Pour cela, il faut lancer le programme et calibrer les 2 caméras pour que les cadrages des 2 caméras soient similaires !


## V) Demo des programmations 

La première chose que je vais faire est de déclarer les bibliothèques nécessaires.

```
import time, board, busio
import numpy as np
import adafruit_mlx90640
import matplotlib.pyplot as plt
import matplotlib.patches as patches
from scipy import ndimage
import cv2
import matplotlib as mpl
mpl.rcParams['toolbar'] = 'None'
```

Les bibliothèques de cartes, busio, ada... sont des bibliothèques pour travailler avec des cartes et des ports d'E/S sur Raspberry Pi.

Ensuite, je vais déclarer la partie reconnaissance faciale ainsi que connecter la webcam USB logitech. J'utilise temporairement HAAR CASCADE pour accélérer le traitement :

```
# Load partie reconnaissance faciale
face_cascade = cv2.CascadeClassifier('haarcascade_frontalface_default.xml')

# Connect camera normal
cap = cv2.VideoCapture(0)
```

Maintenant pour la connexion de la caméra thermique, je vais utiliser le programme suivant :

```
# Connect camera Thermique
i2c = busio.I2C(board.SCL, board.SDA, frequency=100000)
mlx = adafruit_mlx90640.MLX90640(i2c)
mlx.refresh_rate = adafruit_mlx90640.RefreshRate.REFRESH_8_HZ
mlx_shape = (24, 32)

# Dezoomer le matrice thermique
mlx_interp_val = 10  
mlx_interp_shape = (mlx_shape[0] * mlx_interp_val, mlx_shape[1] * mlx_interp_val) 
```

Ici, je fais un peu attention à la variable mlx_shape = (24,32). C'est-à-dire que la taille du capteur de la camera thermique n'est que de 24 × 32, puis elle est interpolée à 240 × 320.

<img width="677" alt="Screen Shot 2020-08-23 at 23 13 54" src="https://user-images.githubusercontent.com/46745468/151548489-000ccc58-62ab-4e6b-a541-d4662a4aa7f3.png">

Vient ensuite le dessin de l'interface principale, constituée de 3 éléments : 
- 1 cellule pour afficher l'image thermique (therm1), 
- 1 cellule pour afficher l'image normale (therm2) 
- et 1 ligne de texte pour afficher la température du visage humain.

```
# Le dessin de l'interface
fig = plt.figure(figsize=(8, 6))  
fig.canvas.set_window_title('Hệ thống giám sát nhiệt độ ra/vào')
fig.canvas.toolbar_visible = False
ax = fig.add_subplot(1, 2, 1)  
ax2 = fig.add_subplot(1, 2, 2)
fig.subplots_adjust(0.05, 0.05, 0.95, 0.95) 

# Afficher l'image thermique
therm1 = ax.imshow(np.zeros(mlx_interp_shape), interpolation='none',
                   cmap=plt.cm.bwr, vmin=25, vmax=45)  # preemptive image

# Afficher l'image normal
therm2 = ax2.imshow(np.zeros((240, 320)), interpolation='none',
                    cmap=plt.cm.bwr, vmin=25, vmax=45)  

fig.canvas.draw()  
ax_background = fig.canvas.copy_from_bbox(ax.bbox)  
fig.show()  
```

Passe maintenant à la partie principale du programme, je vais boucler en continu et lire l'image en même temps, la came principale, la came thermique ... et traiter selon le flux de la section ci-dessus :

```
while True:
    ret, img = cap.read()
    if ret:
        count = count + 1
        img = cv2.resize(img, dsize=(320, 240))
        img2 = cv2.cvtColor(img, cv2.COLOR_BGR2RGB)
        gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
        if count%5==0:
            faces = face_cascade.detectMultiScale(gray, scaleFactor=1.3, minNeighbors=5, minSize=(30, 30))
            if len(faces)==1:
                for (x, y, w, h) in faces:
                    cv2.rectangle(img2, (x, y), (x + w, y + h), (0, 255, 0), 2)
                    try:
                        plot_update(x,y,w,h)  # update plot
                    except:
                        continue
            therm2.set_data(img2)
```
La fonction plot_update, cette fonction se chargera de redessiner les images thermiques, les images bonus et surtout de vérifier la température actuelle ? Si < 38 degrés, c'est normal, s'affiche en jaune et sinon, avertit et s'affiche en rouge vif :

```
# Calculation TEMPERATURE DU FACE
    vface_temp = round(np.max(data_array[y:y+h,x:w+x]), 2)
    if vface_temp<38:
        ax.text(250, -100, 'Nhiệt độ bình thường = ' + str(vface_temp) + "      ",
                bbox={'facecolor': 'yellow', 'alpha': 1, 'pad': 20})
    else:
        ax.text(250, -100, 'Có biểu hiện sốt. Nhiệt độ = ' + str(vface_temp) + "      ",
                bbox={'facecolor': 'red', 'alpha': 1, 'pad': 20})
```

## VI) Resultat

Lors de l'exécution du programme, les résultats de la détection des visages et de la mesure de la température sont assez bons :

![IMG_0044](https://user-images.githubusercontent.com/46745468/151549464-fa8071bb-2b8c-4b18-8c21-0ea88d1047ff.jpg)






