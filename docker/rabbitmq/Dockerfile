FROM rabbitmq:3.13-management

COPY rabbitmq.conf /etc/rabbitmq

RUN rabbitmq-plugins enable rabbitmq_mqtt
RUN rabbitmq-plugins enable rabbitmq_web_mqtt
