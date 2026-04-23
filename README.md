# lab-4
En esta práctica se abordó el estudio de señales electromiográficas (EMG), las cuales permiten analizar la actividad eléctrica generada por los músculos durante procesos de contracción. El propósito principal fue procesar estas señales mediante técnicas de filtrado y análisis en el dominio de la frecuencia, con el fin de identificar indicadores asociados a la fatiga muscular. Para ello, se realizó una comparación entre señales simuladas y señales reales. Adicionalmente, se emplearon herramientas de procesamiento digital para estimar parámetros como la frecuencia media y la frecuencia mediana, evaluando su evolución a lo largo de contracciones sostenidas y su relación con el incremento del esfuerzo muscular.
## Parte A
El primer paso fue configurar el generador de señales para simular la actividad de un músculo mediante una señal EMG. Una vez que obtuvimos esta señal emulada, la capturamos y guardamos para poder trabajar con ella. Después, le aplicamos un proceso de filtrado con el fin de eliminar el ruido y las interferencias, logrando así que la señal se viera de forma clara y nítida para su análisis.
```python
# Librerías:
"""!pip install nidaqmx     
!python -m nidaqmx installdriver """
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
## Parte B
## parte C
