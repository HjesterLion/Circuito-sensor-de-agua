# Circuito-sensor-de-agua
Circuito com sensor de água e notificação via e-mail no thingspeak

        String ssid     = "Simulator Wifi";  // SSID para conectar em um WiFi, neste caso, uma rede "Simulada"
        String password = ""; // Este WiFi não possuí senha

        const int httpPort   = 80;

        String _apiHost        = "api.thingspeak.com";		//URL do Serviço de Nuvem (ThingSpeak)
        String _apiIdWrite  = "6R8990NI4C86UJ4V";		//Colar sua key de escrita - substituindo COLAR_CHAVE_ESCRITA_AQUI
        String _apiRequestUpdate  = "/update?api_key=";
        String _apiField1      = "&field1=";

        String uriWrite     = _apiRequestUpdate + _apiIdWrite + _apiField1; 

        String _apiPathReadChannel = "/channels/";
        String _apiChannelID   = "1235899";		// Colar seu channel ID - substituindo COLAR_CHANNEL_ID_AQUI
        String _apiReadFeed    = "/feeds/last.json?api_key=";	
        String _apiIdRead   = "JTPLW4B69X1TNNV7";		//Colar sua key de leitura - substituindo COLAR_CHAVE_LEITURA_AQUI
        String _apiReadEnd     = "";

        String uriRead     = _apiPathReadChannel + _apiChannelID + _apiReadFeed + _apiIdRead + _apiReadEnd;

        const int GPIO_LED = 12;
        const byte G=3;
        const byte B=4;
        const byte R=5;

        float cm,duracao;

        byte pinoTransmissor=11; // trig
        byte pinoReceptor=10; //echo

        int setupESP8266() {
          Serial.begin(115200);   
          Serial.println("AT");   
          delay(10); 

          if (!Serial.find("OK")) 
            return 1; 

          Serial.println("AT+CWJAP=\"" + ssid + "\",\"" + password + "\"");
          delay(10);       
          if (!Serial.find("OK")) 
            return 2;

          Serial.println("AT+CIPSTART=\"TCP\",\"" + _apiHost + "\"," + httpPort);
          delay(50);      
          if (!Serial.find("OK")) 
            return 3;

          return 0;
        }


        float receberDadosESP8266(){
          String httpPacket3 = "GET " + uriRead + " HTTP/1.1\r\nHost: " + _apiHost + "\r\n\r\n";
          int length3 = httpPacket3.length();

          Serial.print("AT+CIPSEND=");
          Serial.println(length3);
          delay(10); 

          Serial.print(httpPacket3);
          delay(10); 

          while(!Serial.available()) 
            delay(5);	

          String saida = "";

          if (Serial.find("\r\n\r\n")){	
                delay(5);

                unsigned int i = 0; 

                if (!Serial.find("\"field1\":")){}


                        while (i<60000) { 
                    if(Serial.available()) {
                          int c = Serial.read(); 
                          if (c == '.') 
                              break; 
                          if (isDigit(c)) 
                            saida += (char)c; 

                    }
                        i++;
                }
            }
            return saida.toFloat();

        }

        void enviaTemperaturaESP8266() {
          // apenas para limpar o pino transmissor, cortar o sinal e aguardar 5us segundos  
          // (recomendado p/ melhor funcionamento) 
          digitalWrite(pinoTransmissor, LOW);
          delayMicroseconds(5); 
          // envio do pulso de ultrassom 
          digitalWrite(pinoTransmissor, HIGH); 
          // aguarda 10 microsegundos / tempo para o pulso ir e voltar para a leitura
          delayMicroseconds(10);
          // desliga o pino que envia para habiliar o pino que recebe
          digitalWrite(pinoTransmissor, LOW);
          // calcula a duracao em microsegundos do pulso para ir e voltar 
          duracao = pulseIn(pinoReceptor, HIGH);
          // velocidade do som 343 m/s -> 34300 cm / 1000000 us -> 0.00343
          float calcDistancia = (duracao/2) * 0.0343; // em centímetro
          if (calcDistancia>=331){ // fora do limite do sensor
              calcDistancia=0;
          }

          // Construindo a requisição, tipo GET com HTTP
          String httpPacket = "GET " + uriWrite + String(calcDistancia) + " HTTP/1.1\r\nHost: " + _apiHost + "\r\n\r\n";
          int length = httpPacket.length();

          // Enviando tamanho da mensagem por Serial para ESP8266
          Serial.print("AT+CIPSEND=");
          Serial.println(length);
          delay(10);
          // Enviando requisicao/mensagem para ESP8266
          Serial.print(httpPacket);
          delay(10); 
          if (!Serial.find("SEND OK\r\n")) return;
        }

        void acenderLedEmergencia(float pote){
          if(pote >= 100.0){
                digitalWrite(GPIO_LED, HIGH);
          }else{
                digitalWrite(GPIO_LED, LOW);
          }
        }

        float distancia()
        {  
          // apenas para limpar o pino transmissor, cortar o sinal e aguardar 5us segundos  
          // (recomendado p/ melhor funcionamento) 
          digitalWrite(pinoTransmissor, LOW);
          delayMicroseconds(5); 
          // envio do pulso de ultrassom 
          digitalWrite(pinoTransmissor, HIGH); 
          // aguarda 10 microsegundos / tempo para o pulso ir e voltar para a leitura
          delayMicroseconds(10);
          // desliga o pino que envia para habiliar o pino que recebe
          digitalWrite(pinoTransmissor, LOW);
          // calcula a duracao em microsegundos do pulso para ir e voltar 
          duracao = pulseIn(pinoReceptor, HIGH);
          // velocidade do som 343 m/s -> 34300 cm / 1000000 us -> 0.00343
          float calcDistancia= (duracao/2) * 0.0343; // em centímetro
          if (calcDistancia>=331){ // fora do limite do sensor
              calcDistancia=0;
          }
          return calcDistancia;  
        }
        void red(){
            analogWrite(R, 255);
            analogWrite(G, 0);
            analogWrite(B, 0);
          }
        void green(){
            analogWrite(R, 0);
            analogWrite(G, 255);
            analogWrite(B, 0);
          }  
        void yellow(){
                analogWrite(R, 255);
            analogWrite(G, 255);
            analogWrite(B, 0);
          }  
        void apaga(){
            analogWrite(R, 0);
            analogWrite(G, 0);
            analogWrite(B, 0);
          } 

        void setup() {
          pinMode(R, OUTPUT);
          pinMode(G, OUTPUT);
          pinMode(B, OUTPUT);
          pinMode(pinoTransmissor, OUTPUT); // transmissor
          pinMode(pinoReceptor, INPUT);
          setupESP8266();     
        }

        void loop() {
          if(cm >0 && cm<=100){// acende verde
                green();
          }
          else if (cm >100 && cm<=200) { // acende amarelo
                yellow();
          }
          else if (cm>200) { // acende vermelho
                red();
          }
          else {
            apaga();
          }   
          enviaTemperaturaESP8266(); 
          cm =  distancia();
          Serial.print(cm);
          Serial.println(" ml");
          float temperatura = receberDadosESP8266();
          Serial.print(cm);
          acenderLedEmergencia(cm);
          delay(500);
        }
