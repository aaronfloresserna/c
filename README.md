Quiero alinear nuestros tests de Cucumber (legacy) con el wording estándar del proyecto y arreglar los fallos del hook Before.
NO modifiques el pom.xml. Estamos ejecutando con Java 11 y main class legacy cucumber.api.cli.Main.
Contexto actual
Paquete de steps: com.esrx.fhir.observation.service.cucumber.
Los features usan pasos como:
When kafka producer sends the entity message to the ENTITY_TOPIC from file "..."
Then kafka Listener listens the entity message from the ENTITY_TOPIC
And client calls doCustomProcessing with the message … etc.
Error anterior: en @Before se hacía setField(..., "meterRegistry", ...) pero el campo real en ObservationProcessor es meterRegistryUtils.
También hay asserts que comparan JSON como String y fallan por orden de campos.
Tareas
1) Agregar/actualizar escenarios en src/test/resources/ObservationRequestKafka.feature
Pega estos dos escenarios (mismo wording estándar, solo cambian archivos):
# --- Heart Rate ---
Scenario: Test vital-signs Heart Rate observation transformation
  When kafka producer sends the entity message to the ENTITY_TOPIC from file "src/test/resources/ObservationInputHeartRate.json"
  Then kafka Listener listens the entity message from the ENTITY_TOPIC
  And client calls doCustomProcessing with the message
  And receives the joltString as Response
  And client calls parseEncodeToJsonString with the joltString
  And receives the FhirFormat as Response
  And FHIR output should equal to file "src/test/resources/ObservationHeartRateExpected.json"

# --- Body Height ---
Scenario: Test Body Height observation transformation
  When kafka producer sends the entity message to the ENTITY_TOPIC from file "src/test/resources/ObservationInputBodyHeight.json"
  Then kafka Listener listens the entity message from the ENTITY_TOPIC
  And client calls doCustomProcessing with the message
  And receives the joltString as Response
  And client calls parseEncodeToJsonString with the joltString
  And receives the FhirFormat as Response
  And FHIR output should equal to file "src/test/resources/ObservationBodyHeightExpected.json"
(Opcional: etiqueta ambos con @wip para correr solo estos dos.)
2) Arreglar @Before en src/test/java/com/esrx/fhir/observation/service/cucumber/ObservationServiceCucumberTestSteps.java
Crear SimpleMeterRegistry.
Instanciar MeterRegistryUtils sin args y setearle el MeterRegistry por reflexión (campo meterRegistry o registry).
Inyectar meterRegistryUtils al ObservationProcessor (campo exacto: "meterRegistryUtils").
Configurar e inyectar un ObjectMapper y el tagSystem (el literal que está anotado con @Value en el processor).
(Opcional) Mockear LaunchDarklyUtil si es usado en doCustomProcessing.
Snippet a implementar en setup():
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.SerializationFeature;
import org.springframework.test.util.ReflectionTestUtils;
// import org.mockito.Mockito;

SimpleMeterRegistry registry = new SimpleMeterRegistry();

// MeterRegistryUtils sin args + inyección por reflexión
MeterRegistryUtils utils = new MeterRegistryUtils();
boolean injected = false;
try { ReflectionTestUtils.setField(utils, "meterRegistry", registry); injected = true; } catch (IllegalArgumentException ignore) {}
if (!injected) {
  ReflectionTestUtils.setField(utils, "registry", registry);
}
ReflectionTestUtils.setField(observationProcessor, "meterRegistryUtils", utils);

// ObjectMapper consistente con fechas
ObjectMapper mapper = new ObjectMapper()
    .registerModule(new JavaTimeModule())
    .configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
    .configure(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, false);
ReflectionTestUtils.setField(observationProcessor, "mapper", mapper);

// @Value no se inyecta sin Spring context: setear literal del código
ReflectionTestUtils.setField(
    observationProcessor, "tagSystem",
    "urn:uuid:489A4A6A-6B8F-FE61-B6CA-D4160888C790"
);

// (Opcional) Si doCustomProcessing usa LaunchDarklyUtil y da NPE, mocks:
// Object launchDarklyUtil = Mockito.mock(com.esrx.fhir.observation.service.util.LaunchDarklyUtil.class);
// Mockito.when(launchDarklyUtil.isEnabled(Mockito.anyString(), Mockito.anyBoolean())).thenReturn(true);
// ReflectionTestUtils.setField(observationProcessor, "launchDarklyUtil", launchDarklyUtil);
3) Hacer la comparación de FHIR robusta (no String vs String)
En el step que valida el FHIR (método validateFhirFormat, ~línea 459), cambiar el assert para comparar árboles JSON:
import com.fasterxml.jackson.databind.JsonNode;

JsonNode expectedNode = mapper.readTree(expectedStr);
JsonNode actualNode   = mapper.readTree(actualStr);
org.junit.Assert.assertEquals("FHIR JSON no coincide (ignora orden de campos)", expectedNode, actualNode);
(Si el fallo fuese por orden en arrays específicos, ordenar ese array antes de comparar o validar campos clave.)
4) Run configuration (no tocar)
Seguimos ejecutando con la Run Config tipo Application:
Main class: cucumber.api.cli.Main
Program arguments:
--glue com.esrx.fhir.observation.service.cucumber src/test/resources --plugin pretty --plugin html:target/cucumber-report --plugin json:target/cucumber.json
(Para solo estos dos, usar --tags @wip.)
Criterios de aceptación
No vuelve a aparecer el error del hook Before (“Could not find field 'meterRegistry'…”).
Los dos escenarios nuevos corren con el mismo wording estándar de los existentes.
El escenario de Body Height deja de fallar por orden de campos (comparación por JsonNode).
Se genera target/cucumber-report/index.html con resultados de ambos escenarios.
