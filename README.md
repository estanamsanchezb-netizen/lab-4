# lab-4
En esta práctica se abordó el estudio de señales electromiográficas (EMG), las cuales permiten analizar la actividad eléctrica generada por los músculos durante procesos de contracción. El propósito principal fue procesar estas señales mediante técnicas de filtrado y análisis en el dominio de la frecuencia, con el fin de identificar indicadores asociados a la fatiga muscular. Para ello, se realizó una comparación entre señales simuladas y señales reales. Adicionalmente, se emplearon herramientas de procesamiento digital para estimar parámetros como la frecuencia media y la frecuencia mediana, evaluando su evolución a lo largo de contracciones sostenidas y su relación con el incremento del esfuerzo muscular.
## Parte A
El primer paso fue configurar el generador de señales para simular la actividad de un músculo mediante una señal EMG. Una vez que obtuvimos esta señal emulada, la capturamos y guardamos para poder trabajar con ella. Después, le aplicamos un proceso de filtrado con el fin de eliminar el ruido y las interferencias, logrando así que la señal se viera de forma clara y nítida para su análisis.
```python
# Librerías:

import nidaqmx                     # Librería daq. Requiere haber instalado el driver nidaqmx
from nidaqmx.constants import AcquisitionType # Para definir que adquiera datos de manera consecutiva
import matplotlib.pyplot as plt    # Librería para graficar
import numpy as np                 # Librería de funciones matemáticas

#%% Adquisición de la señal por tiempo definido

fs = 2000          # Frecuencia de muestreo en Hz. Recordar cumplir el criterio de Nyquist
duracion = 1    # Periodo por el cual desea medir en segundos
senal = []          # Vector vacío en el que se guardará la señal
dispositivo = 'Dev4/ai0' # Nombre del dispositivo/canal (se puede cambiar el nombre en NI max)

total_muestras = int(fs * duracion)

with nidaqmx.Task() as task:
    # Configuración del canal
    task.ai_channels.add_ai_voltage_chan(dispositivo)
    # Configuración del reloj de muestreo
    task.timing.cfg_samp_clk_timing(
        fs,
        sample_mode=AcquisitionType.FINITE,   # Adquisición finita
        samps_per_chan=total_muestras        # Total de muestras que quiero
    )

    # Lectura de todas las muestras de una vez
    senal = task.read(number_of_samples_per_channel=total_muestras)

t = np.arange(len(senal))/fs # Crea el vector de tiempo 
plt.plot(t,senal)
plt.axis([0,duracion,-6,6])
plt.grid()
plt.title(f"fs={fs}Hz, duración={duracion}s, muestras={len(senal)}")
plt.show()
#%%
np.savetxt(f"labo fs{fs}.txt", [t,senal])
```
Se agarró la señal y se filtró para tener una mejor visualización

```
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.signal import butter, filtfilt, welch

archivo = "/content/captura_señal_fs2000_duracion1 (2).txt"

datos = np.loadtxt(
    archivo,
    comments="#",
    encoding="latin1"
)

t = datos[:, 0]
x = datos[:, 1]

fs = 1 / np.mean(np.diff(t))

print("Frecuencia de muestreo Fs =", fs, "Hz")
print("Periodo de muestreo =", 1/fs, "s")
print("Periodo de muestreo =", (1/fs)*1000, "ms")
print("Duración de la señal =", t[-1] - t[0], "s")
print("Número de muestras =", len(x))

def filtro_pasabanda(signal, fs, f_low=20, f_high=450, orden=4):
    nyquist = fs / 2
    
    low = f_low / nyquist
    high = f_high / nyquist
    
    b, a = butter(
        orden,
        [low, high],
        btype="bandpass"
    )
    
    signal_filtrada = filtfilt(b, a, signal)
    
    return signal_filtrada


x_filtrada = filtro_pasabanda(
    x,
    fs,
    f_low=20,
    f_high=450,
    orden=4
)


plt.figure(figsize=(14, 5))

plt.plot(t, x, label="Señal original", alpha=0.6)
plt.plot(t, x_filtrada, label="Señal filtrada 20-450 Hz", linewidth=1)

plt.xlabel("Tiempo (s)")
plt.ylabel("Amplitud (V)")
plt.title("Señal original vs señal filtrada")
plt.grid(True)
plt.legend()
plt.show()


```
Se segmentó la señal obtenida en las cinco ventanas de 200ms para ser estudiadas

```

duracion_ventana = 0.2   # segundos = 200 ms

muestras_ventana = int(fs * duracion_ventana)
num_ventanas = len(x_filtrada) // muestras_ventana

print("Duración de cada ventana =", duracion_ventana, "s")
print("Duración de cada ventana =", duracion_ventana * 1000, "ms")
print("Muestras por ventana =", muestras_ventana)
print("Número de ventanas completas =", num_ventanas)

ventanas = []

for i in range(num_ventanas):
    inicio = i * muestras_ventana
    fin = inicio + muestras_ventana
    
    t_ventana = t[inicio:fin]
    x_ventana = x_filtrada[inicio:fin]
    
    ventanas.append((t_ventana, x_ventana))


```
Se calculó para cada ventana frecuencia media y frecuencia mediana 

```
def calcular_frecuencias(x_ventana, fs):
   
    
    # Quitar componente DC
    x_ventana = x_ventana - np.mean(x_ventana)
    
    # Calcular densidad espectral de potencia
    f, pxx = welch(
        x_ventana,
        fs=fs,
        nperseg=len(x_ventana)
    )
    
    potencia_total = np.sum(pxx)
    
    if potencia_total == 0:
        return 0, 0
    
    # Frecuencia media
    frecuencia_media = np.sum(f * pxx) / potencia_total
    
    # Frecuencia mediana
    potencia_acumulada = np.cumsum(pxx)
    indice_mediana = np.where(potencia_acumulada >= potencia_acumulada[-1] / 2)[0][0]
    frecuencia_mediana = f[indice_mediana]
    
    return frecuencia_media, frecuencia_mediana



resultados = []

for i, (t_ventana, x_ventana) in enumerate(ventanas):
    
    frecuencia_media, frecuencia_mediana = calcular_frecuencias(
        x_ventana,
        fs
    )
    
    resultados.append({
        "Ventana": i + 1,
        "Tiempo inicial (s)": t_ventana[0],
        "Tiempo final (s)": t_ventana[-1],
        "Frecuencia media (Hz)": frecuencia_media,
        "Frecuencia mediana (Hz)": frecuencia_mediana
    })

tabla_resultados = pd.DataFrame(resultados)

tabla_resultados

from IPython.display import display



tabla_resultados_redondeada = tabla_resultados.copy()

tabla_resultados_redondeada["Tiempo inicial (s)"] = tabla_resultados_redondeada["Tiempo inicial (s)"].round(3)
tabla_resultados_redondeada["Tiempo final (s)"] = tabla_resultados_redondeada["Tiempo final (s)"].round(3)
tabla_resultados_redondeada["Frecuencia media (Hz)"] = tabla_resultados_redondeada["Frecuencia media (Hz)"].round(2)
tabla_resultados_redondeada["Frecuencia mediana (Hz)"] = tabla_resultados_redondeada["Frecuencia mediana (Hz)"].round(2)

display(tabla_resultados_redondeada)


plt.figure(figsize=(12, 5))

plt.plot(
    tabla_resultados["Tiempo inicial (s)"],
    tabla_resultados["Frecuencia media (Hz)"],
    marker="o",
    label="Frecuencia media"
)

plt.plot(
    tabla_resultados["Tiempo inicial (s)"],
    tabla_resultados["Frecuencia mediana (Hz)"],
    marker="s",
    label="Frecuencia mediana"
)

plt.xlabel("Tiempo (s)")
plt.ylabel("Frecuencia (Hz)")
plt.title("Evolución de la frecuencia media y mediana por ventanas de 200 ms")
plt.grid(True)
plt.legend()
plt.show()


plt.figure(figsize=(14, 5))

plt.plot(t, x_filtrada, label="Señal filtrada 20-450 Hz")

for i in range(num_ventanas + 1):
    tiempo_corte = i * duracion_ventana
    plt.axvline(tiempo_corte, linestyle="--", alpha=0.5)

plt.xlabel("Tiempo (s)")
plt.ylabel("Amplitud (V)")
plt.title("Señal filtrada segmentada en ventanas de 200 ms")
plt.grid(True)
plt.legend()
plt.show()

```




## Parte B
Para la adquisición de la señal EMG real se utilizó el sistema BITalino, el cual permite registrar señales bioeléctricas como la actividad muscular. Este dispositivo integra una etapa de acondicionamiento de señales que permite amplificar y filtrar los potenciales bioeléctricos. Su función principal es realizar la conversión analógica-digital (ADC) de señales como la EMG. En este caso, los datos fueron guardados en formato .txt, lo que permitió su posterior visualización y análisis mediante gráficas.
```python
import numpy as np
import matplotlib.pyplot as plt

# Ruta del archivo ya subido
archivo = '/content/señalemg1-04-10_14-28-33.txt'

# Leer datos ignorando encabezado
datos = np.loadtxt(archivo, comments='#')

# Tomar la última columna (A1)
senal = datos[:, -1]

# Frecuencia de muestreo
fs = 1000  # Hz

# Vector de tiempo
tiempo = np.arange(len(senal)) / fs

# Graficar
plt.figure(figsize=(14,4))
plt.plot(tiempo, senal)
plt.title('Señal EMG')
plt.xlabel('Tiempo (s)')
plt.ylabel('Amplitud')
plt.grid(True)
plt.show()
```
<img width="1038" height="328" alt="image" src="https://github.com/user-attachments/assets/1176b669-30d4-4646-ba79-3f446c9d4afc" />



Aplico un filtro pasa banda (20–450 Hz) para eliminar ruido y artefactos. 

```
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from scipy.signal import butter, filtfilt, welch
from IPython.display import display


archivo = "señalemg1-04-10_14-28-33.txt"


datos = np.loadtxt(
    archivo,
    comments="#"
)

# Canal EMG A1 está en la última columna
x = datos[:, 5]

# Según el encabezado del archivo, Fs = 1000 Hz
fs = 1000

# Crear vector de tiempo
t = np.arange(len(x)) / fs

print("Frecuencia de muestreo Fs =", fs, "Hz")
print("Número de muestras =", len(x))
print("Duración de la señal =", t[-1], "s")
print("Primeras muestras EMG:")
print(x[:10])


def filtro_pasabanda(signal, fs, f_low=20, f_high=450, orden=4):
    nyquist = fs / 2
    
    low = f_low / nyquist
    high = f_high / nyquist
    
    b, a = butter(
        orden,
        [low, high],
        btype="bandpass"
    )
    
    signal_filtrada = filtfilt(b, a, signal)
    
    return signal_filtrada


x_filtrada = filtro_pasabanda(
    x_centrada,
    fs,
    f_low=20,
    f_high=450,
    orden=4
)

plt.figure(figsize=(14, 5))
plt.plot(t, x_centrada, label="Señal centrada", alpha=0.5)
plt.plot(t, x_filtrada, label="Señal filtrada 20-450 Hz", linewidth=1)
plt.xlabel("Tiempo (s)")
plt.ylabel("Amplitud")
plt.title("Señal EMG centrada vs señal filtrada")
plt.grid(True)
plt.legend()
plt.show()

```


Se dividio la señal en ventnas de 500ms

```

duracion_ventana = 0.5  # segundos
muestras_ventana = int(fs * duracion_ventana)

num_ventanas = len(x_filtrada) // muestras_ventana

print("Duración de cada ventana =", duracion_ventana, "s")
print("Duración de cada ventana =", duracion_ventana * 1000, "ms")
print("Muestras por ventana =", muestras_ventana)
print("Número de ventanas completas =", num_ventanas)

ventanas = []

for i in range(num_ventanas):
    inicio = i * muestras_ventana
    fin = inicio + muestras_ventana
    
    t_ventana = t[inicio:fin]
    x_ventana = x_filtrada[inicio:fin]
    
    ventanas.append((t_ventana, x_ventana))

```
SE aplico filtro pasa banda (20–450 Hz) para eliminar ruido y artefactos.


```

plt.figure(figsize=(14, 5))

plt.plot(t, x_filtrada, label="Señal EMG filtrada 20-450 Hz")

for i in range(num_ventanas + 1):
    plt.axvline(i * duracion_ventana, linestyle="--", alpha=0.4)

plt.xlabel("Tiempo (s)")
plt.ylabel("Amplitud filtrada")
plt.title("Señal EMG filtrada dividida en ventanas de 500 ms")
plt.grid(True)
plt.legend()
plt.show()

```

Se calculo para cada ventana fq media y mediana

```
def calcular_frecuencias(x_ventana, fs):
    
    # Quitar componente DC residual
    x_ventana = x_ventana - np.mean(x_ventana)
    
    # Densidad espectral de potencia
    f, pxx = welch(
        x_ventana,
        fs=fs,
        nperseg=len(x_ventana)
    )
    
    potencia_total = np.sum(pxx)
    
    if potencia_total == 0:
        return 0, 0, f, pxx
    
    # Frecuencia media
    frecuencia_media = np.sum(f * pxx) / potencia_total
    
    # Frecuencia mediana
    potencia_acumulada = np.cumsum(pxx)
    indice_mediana = np.where(potencia_acumulada >= potencia_total / 2)[0][0]
    frecuencia_mediana = f[indice_mediana]
    
    return frecuencia_media, frecuencia_mediana, f, pxx

resultados = []
espectros = []

for i, (t_ventana, x_ventana) in enumerate(ventanas):
    
    frecuencia_media, frecuencia_mediana, f, pxx = calcular_frecuencias(
        x_ventana,
        fs
    )
    
    resultados.append({
        "Ventana": i + 1,
        "Tiempo inicial (s)": t_ventana[0],
        "Tiempo final (s)": t_ventana[-1],
        "Tiempo centro (s)": (t_ventana[0] + t_ventana[-1]) / 2,
        "Frecuencia media (Hz)": frecuencia_media,
        "Frecuencia mediana (Hz)": frecuencia_mediana
    })
    
    espectros.append((f, pxx))

tabla_resultados = pd.DataFrame(resultados)

tabla_resultados_redondeada = tabla_resultados.round({
    "Tiempo inicial (s)": 3,
    "Tiempo final (s)": 3,
    "Tiempo centro (s)": 3,
    "Frecuencia media (Hz)": 2,
    "Frecuencia mediana (Hz)": 2
})

display(tabla_resultados_redondeada)


```



Se grafican los resultados obtenidos y se analiza la tendencia de la frecuencia
media y mediana a medida que progresa la fatiga muscular. 


```



tiempos_importantes = [4, 40, 85]  

nombres_ventanas = [
    "Inicio de la contracción",
    "Mitad de la contracción",
    "Fatiga final"
]

indices_ventanas_importantes = []

for tiempo_objetivo in tiempos_importantes:
    indice = int(tiempo_objetivo / duracion_ventana)
    
    # Evitar que el índice se salga del rango
    if indice >= num_ventanas:
        indice = num_ventanas - 1
    
    indices_ventanas_importantes.append(indice)

print("Ventanas seleccionadas:")
for nombre, indice in zip(nombres_ventanas, indices_ventanas_importantes):
    print(nombre, "-> Ventana", indice + 1)



# SOBREPOSICIÓN 

plt.figure(figsize=(12, 6))

for nombre, i in zip(nombres_ventanas, indices_ventanas_importantes):
    
    t_ventana, x_ventana = ventanas[i]
    
    # Todas las ventanas empiezan en 0 para compararlas
    tiempo_local = t_ventana - t_ventana[0]
    
    plt.plot(
        tiempo_local,
        x_ventana,
        linewidth=2,
        label=f"{nombre} - Ventana {i+1}"
    )

plt.xlabel("Tiempo dentro de la ventana (s)")
plt.ylabel("Amplitud filtrada")
plt.title("Comparación de ventanas importantes durante la contracción")
plt.grid(True)
plt.legend()
plt.show()


#superposicion espectro

plt.figure(figsize=(12, 6))

for nombre, i in zip(nombres_ventanas, indices_ventanas_importantes):
    
    f, pxx = espectros[i]
    
    plt.plot(
        f,
        pxx,
        linewidth=2,
        label=f"{nombre} - Ventana {i+1}"
    )

plt.xlim(20, 450)
plt.xlabel("Frecuencia (Hz)")
plt.ylabel("Densidad espectral de potencia")
plt.title("Comparación espectral de las ventanas importantes")
plt.grid(True)
plt.legend()
plt.show()


#sobreposicion de las ventanas

plt.figure(figsize=(12, 6))

for i, (f, pxx) in enumerate(espectros):
    plt.plot(
        f,
        pxx,
        alpha=0.45
    )

plt.xlim(20, 450)
plt.xlabel("Frecuencia (Hz)")
plt.ylabel("Densidad espectral de potencia")
plt.title("Espectros de potencia sobrepuestos por ventana")
plt.grid(True)
plt.show()


#evolucion de fq

plt.figure(figsize=(12, 5))

plt.plot(
    tabla_resultados["Tiempo centro (s)"],
    tabla_resultados["Frecuencia media (Hz)"],
    marker="o",
    linewidth=2,
    label="Frecuencia media"
)

plt.plot(
    tabla_resultados["Tiempo centro (s)"],
    tabla_resultados["Frecuencia mediana (Hz)"],
    marker="s",
    linewidth=2,
    label="Frecuencia mediana"
)

plt.xlabel("Tiempo (s)")
plt.ylabel("Frecuencia (Hz)")
plt.title("Evolución de la frecuencia media y mediana durante la fatiga")
plt.grid(True)
plt.legend()
plt.show()
#analisis de tendencia

tiempo = tabla_resultados["Tiempo centro (s)"].values
f_media = tabla_resultados["Frecuencia media (Hz)"].values
f_mediana = tabla_resultados["Frecuencia mediana (Hz)"].values

coef_media = np.polyfit(tiempo, f_media, 1)
coef_mediana = np.polyfit(tiempo, f_mediana, 1)

tendencia_media = np.polyval(coef_media, tiempo)
tendencia_mediana = np.polyval(coef_mediana, tiempo)

pendiente_media = coef_media[0]
pendiente_mediana = coef_mediana[0]

plt.figure(figsize=(12, 5))

plt.plot(tiempo, f_media, marker="o", label="Frecuencia media")
plt.plot(tiempo, tendencia_media, linestyle="--", label=f"Tendencia media: {pendiente_media:.2f} Hz/s")

plt.plot(tiempo, f_mediana, marker="s", label="Frecuencia mediana")
plt.plot(tiempo, tendencia_mediana, linestyle="--", label=f"Tendencia mediana: {pendiente_mediana:.2f} Hz/s")

plt.xlabel("Tiempo (s)")
plt.ylabel("Frecuencia (Hz)")
plt.title("Tendencia de la frecuencia media y mediana durante la fatiga")
plt.grid(True)
plt.legend()
plt.show()

print("Pendiente de frecuencia media:", round(pendiente_media, 4), "Hz/s")
print("Pendiente de frecuencia mediana:", round(pendiente_mediana, 4), "Hz/s")



```


<img width="117" height="682" alt="image" src="https://github.com/user-attachments/assets/6e486d31-f9a1-49b7-af9b-8a9867ce060e" />


## parte C

Se le aplica la la Transformada Rápida de Fourier (FFT) a cada contracción de la
señal EMG real.

```


tiempo_inicio_contraccion = 3  #

indice_inicio_contraccion = int(tiempo_inicio_contraccion / duracion_ventana)

print("La contracción empieza aproximadamente en la ventana:", indice_inicio_contraccion + 1)

resultados_fft = []
fft_ventanas = []

for i in range(indice_inicio_contraccion, num_ventanas):
    
    t_ventana, x_ventana = ventanas[i]
  
    x_temp = x_ventana - np.mean(x_ventana)
    
    # Número de muestras
    N = len(x_temp)
    
   
    fft_valores = np.fft.rfft(x_temp)
    frecuencias_fft = np.fft.rfftfreq(N, d=1/fs)
   
    magnitud = np.abs(fft_valores) / N
    
  
    mascara = (frecuencias_fft >= 20) & (frecuencias_fft <= 450)
    
    f_util = frecuencias_fft[mascara]
    mag_util = magnitud[mascara]
  
    indice_pico = np.argmax(mag_util)
    pico_frecuencia = f_util[indice_pico]
    pico_magnitud = mag_util[indice_pico]
    
 
    umbral_alta_frecuencia = 150  # Hz
    
    mascara_alta = f_util >= umbral_alta_frecuencia
    energia_total = np.sum(mag_util**2)
    energia_alta = np.sum(mag_util[mascara_alta]**2)
    
    porcentaje_alta_frecuencia = (energia_alta / energia_total) * 100 if energia_total != 0 else 0
    
    resultados_fft.append({
        "Ventana": i + 1,
        "Tiempo inicial (s)": t_ventana[0],
        "Tiempo final (s)": t_ventana[-1],
        "Tiempo centro (s)": (t_ventana[0] + t_ventana[-1]) / 2,
        "Pico espectral (Hz)": pico_frecuencia,
        "Magnitud pico": pico_magnitud,
        "Contenido alta frecuencia (%)": porcentaje_alta_frecuencia
    })
    
    fft_ventanas.append({
        "indice_ventana": i,
        "frecuencias": f_util,
        "magnitud": mag_util,
        "pico_frecuencia": pico_frecuencia,
        "pico_magnitud": pico_magnitud,
        "contenido_alta": porcentaje_alta_frecuencia
    })

tabla_fft = pd.DataFrame(resultados_fft)

tabla_fft_redondeada = tabla_fft.round({
    "Tiempo inicial (s)": 3,
    "Tiempo final (s)": 3,
    "Tiempo centro (s)": 3,
    "Pico espectral (Hz)": 2,
    "Magnitud pico": 5,
    "Contenido alta frecuencia (%)": 2
})

display(tabla_fft_redondeada)

plt.figure(figsize=(12, 6))

for item in fft_ventanas:
    plt.plot(
        item["frecuencias"],
        item["magnitud"],
        alpha=0.35
    )

plt.xlim(20, 450)
plt.xlabel("Frecuencia (Hz)")
plt.ylabel("Magnitud")
plt.title("Espectros de amplitud FFT por ventana desde el inicio de la contracción")
plt.grid(True)
plt.show()


```
 se comparan las primeras ventanas con las ultimas para ver el cmbio con la fatiga


 ```

cantidad_ventanas_comparar = 3

primeras_fft = fft_ventanas[:cantidad_ventanas_comparar]
ultimas_fft = fft_ventanas[-cantidad_ventanas_comparar:]

plt.figure(figsize=(12, 6))

# Primeras ventanas
for item in primeras_fft:
    plt.plot(
        item["frecuencias"],
        item["magnitud"],
        linewidth=2,
        linestyle="-",
        label=f"Inicio - Ventana {item['indice_ventana'] + 1}"
    )

# Últimas ventanas
for item in ultimas_fft:
    plt.plot(
        item["frecuencias"],
        item["magnitud"],
        linewidth=2,
        linestyle="--",
        label=f"Fatiga - Ventana {item['indice_ventana'] + 1}"
    )

plt.xlim(20, 450)
plt.xlabel("Frecuencia (Hz)")
plt.ylabel("Magnitud")
plt.title("Comparación FFT: primeras ventanas vs últimas ventanas")
plt.grid(True)
plt.legend()
plt.show()



frecuencias_ref = primeras_fft[0]["frecuencias"]

magnitudes_inicio = np.array([item["magnitud"] for item in primeras_fft])
magnitudes_fatiga = np.array([item["magnitud"] for item in ultimas_fft])

promedio_inicio = np.mean(magnitudes_inicio, axis=0)
promedio_fatiga = np.mean(magnitudes_fatiga, axis=0)

plt.figure(figsize=(12, 6))

plt.plot(
    frecuencias_ref,
    promedio_inicio,
    linewidth=3,
    label="Promedio primeras ventanas"
)

plt.plot(
    frecuencias_ref,
    promedio_fatiga,
    linewidth=3,
    linestyle="--",
    label="Promedio últimas ventanas / fatiga"
)

plt.xlim(20, 450)
plt.xlabel("Frecuencia (Hz)")
plt.ylabel("Magnitud promedio")
plt.title("Comparación del contenido frecuencial: inicio vs fatiga")
plt.grid(True)
plt.legend()
plt.show()


pico_inicio = np.mean([item["pico_frecuencia"] for item in primeras_fft])
pico_fatiga = np.mean([item["pico_frecuencia"] for item in ultimas_fft])

desplazamiento_pico = pico_fatiga - pico_inicio

print("Pico espectral promedio al inicio:", round(pico_inicio, 2), "Hz")
print("Pico espectral promedio en fatiga:", round(pico_fatiga, 2), "Hz")
print("Desplazamiento del pico espectral:", round(desplazamiento_pico, 2), "Hz"


plt.figure(figsize=(12, 5))

plt.plot(
    tabla_fft["Tiempo centro (s)"],
    tabla_fft["Pico espectral (Hz)"],
    marker="o",
    linewidth=2,
    label="Pico espectral"
)

plt.xlabel("Tiempo (s)")
plt.ylabel("Frecuencia del pico espectral (Hz)")
plt.title("Evolución del pico espectral durante la contracción")
plt.grid(True)
plt.legend()
plt.show()

#reduccion de alta fq

plt.figure(figsize=(12, 5))

plt.plot(
    tabla_fft["Tiempo centro (s)"],
    tabla_fft["Contenido alta frecuencia (%)"],
    marker="s",
    linewidth=2,
    label="Contenido >150 Hz"
)

plt.xlabel("Tiempo (s)")
plt.ylabel("Contenido de alta frecuencia (%)")
plt.title("Evolución del contenido de alta frecuencia durante la fatiga")
plt.grid(True)
plt.legend()
plt.show()

#tabla incio vs fatiga

alta_inicio = np.mean([item["contenido_alta"] for item in primeras_fft])
alta_fatiga = np.mean([item["contenido_alta"] for item in ultimas_fft])

resumen_fft = pd.DataFrame({
    "Condición": ["Inicio de la contracción", "Fatiga final"],
    "Pico espectral promedio (Hz)": [pico_inicio, pico_fatiga],
    "Contenido alta frecuencia promedio (%)": [alta_inicio, alta_fatiga]
})

resumen_fft["Pico espectral promedio (Hz)"] = resumen_fft["Pico espectral promedio (Hz)"].round(2)
resumen_fft["Contenido alta frecuencia promedio (%)"] = resumen_fft["Contenido alta frecuencia promedio (%)"].round(2)

display(resumen_fft)


```
<img width="101" height="898" alt="image" src="https://github.com/user-attachments/assets/8cc752b7-cdf1-42fd-b51e-fd2b0821a420" />
