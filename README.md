<p align="center">
  <img src="https://1.bp.blogspot.com/-CH8TRL_RVUM/VscQfCRByTI/AAAAAAAAJW0/RLeLd2w1LbU/s1600/Work_In_Progress.png" width="175" height="150">
</p>

# Projeto - Conectando um Fiat Palio na AWS

No dia 06/05/2020, Vinicius Senger (Senior Technical Evangelist na Amazon Web Services) participou do 4º Open Space da DataSprints, onde este apresentou um projeto bem peculiar, que foi como conectar uma Kombi na Nuvem!

[<img src="Screenshot1.png">](https://www.youtube.com/watch?v=tqBR3G_eFQc&t=2417s)
<p align="center">
<sub>Para abrir o vídeo no Youtube, clique na imagem!</sub>
</p>

---

## Conectando meu carro na nuvem!
Como proprietário de um Fiat Palio 2014/2015, me senti motivado a tentar conectá-lo na nuvem também, iniciando um novo projeto.

Para ainda mais informações sobre o processo de receber as informações do veículo e transmití-las à AWS, o Vinicius Senger possui publicado em seu canal do Youtube mais um vídeo.

[<img src="Screenshot2.png">](https://www.youtube.com/watch?v=Iiku8HAPAgo&t=316s)
<p align="center">
<sub>Para abrir o vídeo no Youtube, clique na imagem!</sub>
</p> 

Basicamente, o fluxo deve ser:
**OBDII** -> **Laptop** -> **AWS**

Diferentemente do apresentado pelo Vinicius, que utiliza um módulo de comunicação OBDII USB, para este projeto, irei utilizar um módulo OBDII com conexão bluetooth.

<p align="center">
  <img src="OBDII.png" width="200" height="200">
</p>

Para o Fiat Palio, a entrada do OBDII se localiza em um compartimento abaixo do volante. É necessário remover um parafuso para ter acesso à entrada.

<p align="center">
  <img src="Entrada3.png" >
</p>

<p align="center">
  <img src="Entrada.png" width="150" height="150">
</p>

<p align="center">
  <img src="Entrada2.png" width="150" height="150">
</p>

Conectado o módulo OBD2 no carro, e após pareá-lo com o laptop, é necessário identificar o serial do módulo bluetooth (no meu caso, tty63) e rodar o código:

```import serial
import time
ser = serial.Serial('/dev/tty63', 38400, timeout=1, bytesize=8, parity=serial.PARITY_NONE)
time.sleep(1)
ser.write("atz\r\n".encode())
print(ser.readline())
rpm = 0
speed = 0
temp = 0
initializing = True
time.sleep(1)

# Import SDK packages
from AWSIoTPythonSDK.MQTTLib import AWSIoTMQTTClient

myMQTTClient = AWSIoTMQTTClient("myClientID-Fiat_Palio")
myMQTTClient.configureEndpoint("ahjysgxc7yc9n-ats.iot.us-west-2.amazonaws.com", 8883)
myMQTTClient.configureCredentials("/home/lucasmirachi/Documents/Car_Reader/connect_device_package/root-CA.crt", "/home/lucasmirachi/Documents/Car_Reader/connect_device_package/Fiat_Palio.private.key", "/home/lucasmirachi/Documents/Car_Reader/connect_device_package/Fiat_Palio.cert.pem")
myMQTTClient.configureOfflinePublishQueueing(-1)  # Infinite offline Publish queueing
myMQTTClient.configureDrainingFrequency(2)  # Draining: 2 Hz
myMQTTClient.configureConnectDisconnectTimeout(10)  # 10 sec
myMQTTClient.configureMQTTOperationTimeout(5)  # 5 sec
myMQTTClient.connect()


while initializing:
	ser.write("010c\r\n".encode())
	r = ser.readline();
	if(r.startswith("010c")):
		initializing= False
	else:
		time.sleep(2)

while True:
	ser.write("010c\r\n".encode())
	rpm_h = ser.readline()
	#print(rpm_h)
	#print(len(rpm_h))
	if(rpm_h.startswith("010c")):
		try:
			rpm_1 = rpm_h[11:13]
			rpm_2 = rpm_h[14:16]
			rpm = int(rpm_1 + rpm_2, 16)/4
			print("rpm " + str(rpm))
		except ValueError:
			print("valor errado")
	else:
		print("bus initializing")
	time.sleep(1)
	ser.write("010d\r\n".encode())
	speed_h = ser.readline()
	#print(speed_h)
	#print(len(speed_h))
	if(speed_h.startswith("010d")):
		try:
			speed_1 = speed_h[11:13]
			speed = int(speed_1, 16)
			print("speed " + str(speed))
		except ValueError:
			print("valor errado")
	else:
		print("bus initializing.")
	time.sleep(1)

	ser.write("0105\r\n".encode())
	temp_h = ser.readline()
	#print(speed_h)
	#print(len(speed_h))
	if(temp_h.startswith("0105")):
		try:
			temp_1 = temp_h[11:13]
			temp = int(temp_1, 16) - 40
			print("temperature " + str(temp))
		except ValueError:
			print("valor errado")
	else:
		print("bus initializing...")


	myMQTTClient.publish("connected_vehicle",'{ "rpm" : ' + str(rpm) + ', "speed" : ' + str(speed) + ',"temperature : ' + str(temp) + '}', 0)
	time.sleep(1)```
  
