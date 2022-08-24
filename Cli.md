# Juste simple CLI all-the-way-down ;-)

## create project

```shell
mvn io.quarkus.platform:quarkus-maven-plugin:2.11.2.Final:create \
    -DprojectGroupId=io.github.kandefromparis.khool \
    -DprojectArtifactId=kafka-quickstart-producer \
    -Dextensions="resteasy-reactive-jackson,smallrye-reactive-messaging-kafka" \
    -DnoCode

    
mvn io.quarkus.platform:quarkus-maven-plugin:2.11.2.Final:create \
    -DprojectGroupId=io.github.kandefromparis.khool \
    -DprojectArtifactId=kafka-quickstart-processor \
    -Dextensions="smallrye-reactive-messaging-kafka" \
    -DnoCode
    
```

## create quote.java

```shell
mkdir -p ./kafka-quickstart-{processor,producer}/src/main/java/io/github/kandefromparis/khool/kafka/{model,processor,producer}/    

cat << EOF > ./kafka-quickstart-processor/src/main/java/io/github/kandefromparis/khool/kafka/model/Quote.java
package io.github.kandefromparis.khool.kafka.model;

public class Quote {

    public String id;
    public int price;

    /**
    * Default constructor required for Jackson serializer
    */
    public Quote() { }

    public Quote(String id, int price) {
        this.id = id;
        this.price = price;
    }

    @Override
    public String toString() {
        return "Quote{" +
                "id='" + id + '\'' +
                ", price=" + price +
                '}';
    }
}
EOF

cat << EOF > ./kafka-quickstart-producer/src/main/java/io/github/kandefromparis/khool/kafka/model/Quote.java
package io.github.kandefromparis.khool.kafka.model;

public class Quote {

    public String id;
    public int price;

    /**
    * Default constructor required for Jackson serializer
    */
    public Quote() { }

    public Quote(String id, int price) {
        this.id = id;
        this.price = price;
    }

    @Override
    public String toString() {
        return "Quote{" +
                "id='" + id + '\'' +
                ", price=" + price +
                '}';
    }
}
EOF
```

```shell
cat << EOF > ./kafka-quickstart-producer/src/main/java/io/github/kandefromparis/khool/kafka/producer/QuotesResource.java
package io.github.kandefromparis.khool.kafka.producer;

import java.util.UUID;

import javax.ws.rs.GET;
import javax.ws.rs.POST;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import io.github.kandefromparis.khool.kafka.model.Quote;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;

@Path("/quotes")
public class QuotesResource {

    @Channel("quote-requests")
    Emitter<String> quoteRequestEmitter; 

    /**
     * Endpoint to generate a new quote request id and send it to "quote-requests" Kafka topic using the emitter.
     */
    @POST
    @Path("/request")
    @Produces(MediaType.TEXT_PLAIN)
    public String createRequest() {
        UUID uuid = UUID.randomUUID();
        quoteRequestEmitter.send(uuid.toString()); 
        return uuid.toString(); 
    }
}
EOF
```

```shell
cat << EOF > ./kafka-quickstart-processor/src/main/java/io/github/kandefromparis/khool/kafka/processor/QuotesProcessor.java
package io.github.kandefromparis.khool.kafka.processor;

import java.util.Random;

import javax.enterprise.context.ApplicationScoped;

import io.github.kandefromparis.khool.kafka.model.Quote;
import org.eclipse.microprofile.reactive.messaging.Incoming;
import org.eclipse.microprofile.reactive.messaging.Outgoing;

import io.smallrye.reactive.messaging.annotations.Blocking;

/**
 * A bean consuming data from the "quote-requests" Kafka topic (mapped to "requests" channel) and giving out a random quote.
 * The result is pushed to the "quotes" Kafka topic.
 */
@ApplicationScoped
public class QuotesProcessor {

    private Random random = new Random();

    @Incoming("requests") 
    @Outgoing("quotes")   
    @Blocking             
    public Quote process(String quoteRequest) throws InterruptedException {
        // simulate some hard working task
        Thread.sleep(200);
        return new Quote(quoteRequest, random.nextInt(100));
    }
}

EOF
```

```shell
rm ./kafka-quickstart-processor/src/main/resources/application.properties
cat << EOF > ./kafka-quickstart-processor/src/main/resources/application.properties

%dev.quarkus.http.port=8081

# Configure the incoming \`quote-requests\` Kafka topic
mp.messaging.incoming.requests.topic=quote-requests
mp.messaging.incoming.requests.auto.offset.reset=earliest
EOF
```

```shell
rm ./kafka-quickstart-producer/src/main/java/io/github/kandefromparis/khool/kafka/producer/QuotesResource.java;
cat << EOF > ./kafka-quickstart-producer/src/main/java/io/github/kandefromparis/khool/kafka/producer/QuotesResource.java
package io.github.kandefromparis.khool.kafka.producer;

import java.util.UUID;

import javax.ws.rs.GET;
import javax.ws.rs.POST;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import io.smallrye.mutiny.Multi;

import io.github.kandefromparis.khool.kafka.model.Quote;
import org.eclipse.microprofile.reactive.messaging.Channel;
import org.eclipse.microprofile.reactive.messaging.Emitter;

@Path("/quotes")
public class QuotesResource {

    @Channel("quote-requests")
    Emitter<String> quoteRequestEmitter; 

    /**
     * Endpoint to generate a new quote request id and send it to "quote-requests" Kafka topic using the emitter.
     */
    @POST
    @Path("/request")
    @Produces(MediaType.TEXT_PLAIN)
    public String createRequest() {
        UUID uuid = UUID.randomUUID();
        quoteRequestEmitter.send(uuid.toString()); 
        return uuid.toString(); 
    }
    
    @Channel("quotes")
    Multi<Quote> quotes;

    /**
     * Endpoint retrieving the "quotes" Kafka topic and sending the items to a server sent event.
     */
    @GET
    @Produces(MediaType.SERVER_SENT_EVENTS)
    public Multi<Quote> stream() {
        return quotes;
    }
}
EOF
```

```shell
mkdir -p ./kafka-quickstart-producer/src/main/resources/META-INF/resources;
cat << EOF > ./kafka-quickstart-producer/src/main/resources/META-INF/resources/quotes.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Prices</title>

    <link rel="stylesheet" type="text/css"
          href="https://cdnjs.cloudflare.com/ajax/libs/patternfly/3.24.0/css/patternfly.min.css">
    <link rel="stylesheet" type="text/css"
          href="https://cdnjs.cloudflare.com/ajax/libs/patternfly/3.24.0/css/patternfly-additions.min.css">
</head>
<body>
<div class="container">
    <div class="card">
        <div class="card-body">
            <h2 class="card-title">Quotes</h2>
            <button class="btn btn-info" id="request-quote">Request Quote</button>
            <div class="quotes"></div>
        </div>
    </div>
</div>
</body>
<script src="https://code.jquery.com/jquery-3.6.0.min.js"></script>
<script>
    \$("#request-quote").click((event) => {
        fetch("/quotes/request", {method: "POST"})
        .then(res => res.text())
        .then(qid => {
            var row = \$(\`<h4 class='col-md-12' id='\${qid}'>Quote # <i>\${qid}</i> | <strong>Pending</strong></h4>\`);
            \$(".quotes").prepend(row);
        });
    });

    var source = new EventSource("/quotes");
    source.onmessage = (event) => {
      var json = JSON.parse(event.data);
      \$(\`#\${json.id}\`).html((index, html) => {
        return html.replace("Pending", \`\\\$\\xA0\${json.price}\`);
      });
    };
</script>
</html>
EOF
```

## start minikube in order to use Docker

```shell
minikube -p khool-kogito start;
eval $(minikube -p khool-kogito docker-env);
docker pull docker.io/vectorized/redpanda:v21.11.3.
```

## start each project in its own terminal

```shell
mvn -f kafka-quickstart-producer quarkus:dev 
```

```shell
mvn -f kafka-quickstart-processor quarkus:dev
```

```shell
open http://localhost:8080/quotes.html
```
