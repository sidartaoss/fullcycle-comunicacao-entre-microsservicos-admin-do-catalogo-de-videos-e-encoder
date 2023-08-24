# Comunicação entre os Microsserviços de Administração do Catálogo de Vídeos e Encoder

1. Criar a rede externa _adm_videos_services_:

```bash
docker network create "adm_videos_services"
```

2. Neste caso, é defindo o serviço do _RabbitMQ_ no arquivo _docker-compose.yaml_ da aplicação _adm-videos_ para ser acessado a partir da rede externa _adm_videos_services_:

```text
version: '3.7'

services:
  mysql:
    container_name: adm_videos_mysql
    image: mysql:latest
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=adm_videos
    security_opt:
      - seccomp:unconfined
    ports:
      - 33306:3306
    networks:
      - adm_videos_network

  rabbitmq:
    container_name: adm_videos_rabbitmq
    image: rabbitmq:3-management
    environment:
      - RABBITMQ_ERLANG_COOKIE=SWQOKODSQALRPCLNMEQG
      - RABBITMQ_DEFAULT_USER=adm_videos
      - RABBITMQ_DEFAULT_PASS=123456
      - RABBITMQ_DEFAULT_VHOST=/
    ports:
      - 15672:15672
      - 5672:5672
    networks:
      - adm_videos_services

networks:
  adm_videos_network:
  adm_videos_services:
    external: true
```

3. A aplicação _encoder_ deve fazer referência à rede externa em seu arquivo _docker-compose.yaml_ para ter acesso ao _RabbitMQ_:

```text
version: "3"

services:
  app:
    build: .
    volumes:
      - .:/go/src
    networks:
      - encoder_network
      - adm_videos_services

  db:
    image: postgres:9.4
    restart: always
    tty: true
    volumes:
      - .pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=encoder
    ports:
      - "5432:5432"
    networks:
      - encoder_network

  # rabbit:
  #   image: "rabbitmq:3-management"
  #   environment:
  #     RABBITMQ_ERLANG_COOKIE: "SWQOKODSQALRPCLNMEQG"
  #     RABBITMQ_DEFAULT_USER: "rabbitmq"
  #     RABBITMQ_DEFAULT_PASS: "rabbitmq"
  #     RABBITMQ_DEFAULT_VHOST: "/"
  #   ports:
  #     - "15672:15672"
  #     - "5672:5672"


networks:
  encoder_network:
  adm_videos_services:
    external: true
```

4. No arquivo _.env_ da aplicação _encoder_:

- Alterar os dados de conexão do _RabbitMQ_ conforme definido no Passo 1:

```text
RABBITMQ_DEFAULT_USER=adm_videos
RABBITMQ_DEFAULT_PASS=123456
RABBITMQ_DEFAULT_HOST=rabbitmq
```

- Alterar os dados de fila, _exchange_ e _routing key_, conforme definidos na aplicação _adm-videos_:

```text
RABBITMQ_CONSUMER_QUEUE_NAME=video.created.queue
RABBITMQ_NOTIFICATION_EX=video.events
RABBITMQ_NOTIFICATION_ROUTING_KEY=video.encoded
```

O atributo _RABBITMQ_CONSUMER_QUEUE_NAME_ refere-se à fila para onde a aplicação _adm-videos_ envia a notificação no formato _JSON_ após fazer o _upload_ de um arquivo de vídeo para o _bucket_ do _Google Cloud Storage_. A notificação é consumida pela aplicação _encoder_.

Exemplo de notificação enviada pela aplicação _adm-videos_:

```text
{
   "resource_id":"c7a9ee42a7f44ecd96bc10a660c794e9",
   "file_path":"videoId-696d02e507824064994be29b4b845214/type-VIDEO",
   "occurred_on":"2023-08-23T19:27:51.197930Z"
}
```

Após finalizar o processamento de _encoding_ de vídeo, a aplicação _encoder_ utiliza o valor dos atributos _RABBITMQ_NOTIFICATION_EX_ e _RABBITMQ_NOTIFICATION_ROUTING_KEY_ para publicar uma notificação para a aplicação _adm-videos_ consumir para atualizar o banco de dados com o _status_ do processamento.

Exemplo de notificação enviada pela aplicação _encoder_:

```text
{
   "job_id":"f948a3bf-64d7-4647-a722-9ef782135d5d",
   "output_bucket_path":"fc3_catalogo_videos_sidarta_silva",
   "status":"COMPLETED",
   "video":{
      "encoded_video_folder":"15637671-4e0d-4740-863e-4009f368c75d",
      "resource_id":"921bbc9cd531485e8a9acfc4b849e8b1",
      "file_path":"videoId-3b1736d656d64c76b523a825c43c111c/type-VIDEO"
   },
   "Error":"",
   "created_at":"2023-08-23T05:16:43.17475844Z",
   "updated_at":"2023-08-23T05:17:00.711703764Z"
}
```

5. Na aplicação _adm-videos_:

- Na classe _domain/src/main/java/com/fullcycle/admin/catalogo/domain/video/Video.java_:

Passar _AudioVideoMedia.id()_ como _resourceId_ no construtor da classe _VideoMediaCreated_:

```text
    private void onAudioVideoUpdated(AudioVideoMedia audioVideoMedia) {
        if (audioVideoMedia != null && audioVideoMedia.isPendingEncode()) {
            this.registerEvent(new VideoMediaCreated(audioVideoMedia.id(), audioVideoMedia.rawLocation()));
        }
    }
```

- Na classe _infrastructure/src/main/java/com/fullcycle/admin/catalogo/infrastructure/configuration/AmqpConfig.java_:

a. Alterar a definição do _Bean_ _VideoCreatedQueue_ para incluir a definição de um argumento de _x-dead-letter-exchange_, ou seja, no caso de erro, uma _dead-letter-exchange_ para onde encaminhar a mensagem:

```text
        @Bean
        @VideoCreatedQueue
        public Queue videoCreatedQueue(@VideoCreatedQueue QueueProperties props) {
            return QueueBuilder.durable(props.getQueue())
                    .withArgument("x-dead-letter-exchange", "dlx")
                    .build();
//            return new Queue(props.getQueue());
        }
```

Lembrando que a definição de um argumento de _x-dead-letter-exchange_ é feita também pela aplicação _encoder_, ou seja, a aplicação _Golang_ espera pela configuração desse argumento para a fila _video.created.queue_.

b. Para o Bean _VideoEncodedQueue_, também pode-se incluir a definição de uma _x-dead-letter-exchange_:

```text
        @Bean
        @VideoEncodedQueue
        public Queue videoEncodedQueue(@VideoEncodedQueue QueueProperties props) {
            return QueueBuilder.durable(props.getQueue())
                    .withArgument("x-dead-letter-exchange", "dlx")
                    .build();
        }
```

Lembrando que deve-se criar uma nova _Exchange_ no painel de controle do _RabbitMQ_ com o nome de _dlx_, do tipo _fanout_, vinculando a uma nova fila com o nome de _videos.failed_, por exemplo.
  
c. Incluir o _Bean_ _rabbitListenerContainerFactory_ para configurar o conversor de mensagem do _Jackson_, o _JavaTimeModule_ e o _AcknowledgeMode_ do _RabbitMQ_ para manual:

```text
    @Bean("rabbitListenerContainerFactory")
    public RabbitListenerContainerFactory<?> rabbitFactory
            (ConnectionFactory connectionFactory) {
        var factory = new SimpleRabbitListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        ObjectMapper objectMapper = new Jackson2ObjectMapperBuilder()
                .dateFormat(new StdDateFormat())
                .modules(new JavaTimeModule(), new Jdk8Module())
                .build();
        factory.setMessageConverter(new Jackson2JsonMessageConverter(objectMapper));
        factory.setDefaultRequeueRejected(false);
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);
        return factory;
    }
```

- No record _infrastructure/src/main/java/com/fullcycle/admin/catalogo/infrastructure/video/models/VideoEncoderCompleted.java_:

Alterar os atributos conforme:

```
@JsonTypeName("COMPLETED")
public record VideoEncoderCompleted(
        @JsonProperty("job_id") String id,
        @JsonProperty("output_bucket_path") String outputBucket,
        @JsonProperty("video") VideoMetadata video,
        @JsonProperty("Error") String error,
        @JsonProperty("created_at") Instant createdAt,
        @JsonProperty("updated_at") Instant updatedAt

        ) implements VideoEncoderResult {
```

- Na classe _infrastructure/src/main/java/com/fullcycle/admin/catalogo/infrastructure/amqp/VideoEncoderListener.java_:

Alterar o método _onVideoEncodedMessage_ conforme:

```text
import org.springframework.amqp.support.AmqpHeaders;
import org.springframework.messaging.handler.annotation.Header;
import com.rabbitmq.client.Channel;

@RabbitListener(id = LISTENER_ID, queues = "${amqp.queues.video-encoded.queue}")
    public void onVideoEncodedMessage(@Payload final VideoEncoderResult message,
                                      Channel channel,
                                      @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        if (message instanceof VideoEncoderCompleted dto) {
            log.info("[message:video.listener.income] [status:completed] [payload:{}]", message);
            final var aCommand =
                    UpdateMediaStatusCommand.with(
                            MediaStatus.COMPLETED,
                            dto.id(),
                            dto.video().resourceId(),
                            dto.video().encodedVideoFolder(),
                            dto.video().filePath());
            this.updateMediaStatusUseCase.execute(aCommand);
            channel.basicAck(tag, false);
        } else if (message instanceof VideoEncoderError) {
            log.error("[message:video.listener.income] [status:error] [payload:{}]", message);
        } else {
            log.error("[message:video.listener.income] [status:unknown] [payload:{}]", message);
        }
    }
```

Dessa forma, é possível retornar um _Ack_ para a fila somente após o processamento ter sido executado com sucesso.

- Na classe _application/src/main/java/com/fullcycle/admin/catalogo/application/video/media/update/DefaultUpdateMediaStatusUseCase.java_:

É recebido um _payload_ da aplicação _encoder_ no formato:

```text
{
   "job_id":"f948a3bf-64d7-4647-a722-9ef782135d5d",
   "output_bucket_path":"fc3_catalogo_videos_sidarta_silva",
   "status":"COMPLETED",
   "video":{
      "encoded_video_folder":"15637671-4e0d-4740-863e-4009f368c75d",
      "resource_id":"921bbc9cd531485e8a9acfc4b849e8b1",
      "file_path":"videoId-3b1736d656d64c76b523a825c43c111c/type-VIDEO"
   },
   "Error":"",
   "created_at":"2023-08-23T05:16:43.17475844Z",
   "updated_at":"2023-08-23T05:17:00.711703764Z"
}
```

Aonde _videoId_ encontra-se no valor do atributo _file_path_. Então, criar o método privado _videoIdOf_:

```text
    private String videoIdOf(final String filePath) {
        final var beginIndex = filePath.indexOf('-');
        final var endIndex = filePath.indexOf('/');
        return filePath.substring(beginIndex + 1, endIndex);
    }
```

E obter o _videoId_ do objeto de comando a partir do atributo _filename_:

```text
 @Override
    public void execute(UpdateMediaStatusCommand aCommand) {
        final var anId = videoIdOf(aCommand.filename());
        final var aVideoId = VideoID.from(anId);
```
