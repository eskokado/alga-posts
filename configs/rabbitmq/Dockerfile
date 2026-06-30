FROM rabbitmq:3-management

RUN rabbitmq-plugins enable --offline \
    rabbitmq_management \
    rabbitmq_shovel \
    rabbitmq_shovel_management

COPY rabbitmq.conf /etc/rabbitmq/rabbitmq.conf
