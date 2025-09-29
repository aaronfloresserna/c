Prompt para Cursor:
Quiero ejecutar los tests de Cucumber (legacy) en el proyecto fhir-hl-observation-transformer con Java 11 en Windows/IntelliJ. Ya corren vía Run Configuration → Application usando la main class legacy cucumber.api.cli.Main y los features en src/test/resources.
El reporte target/cucumber-report/index.html falla en el hook Before con este error repetido:
java.lang.IllegalArgumentException: Could not find field 'meterRegistry' of type [null]
at org.springframework.test.util.ReflectionTestUtils.setField(ReflectionTestUtils.java:...)
at com.esrx.fhir.observation.service.cucumber.ObservationServiceCucumberTestSteps.setup(ObservationServiceCucumberTestSteps.java:169)
Revisando el código, en src/main/java/com/esrx/fhir/observation/service/ObservationProcessor.java no existe el campo meterRegistry. El campo real es:
@Autowired
MeterRegistryUtils meterRegistryUtils;
Tarea
Arregla el hook @Before en src/test/java/com/esrx/fhir/observation/service/cucumber/ObservationServiceCucumberTestSteps.java para que deje de hacer setField(..., "meterRegistry", ...) y, en su lugar, inyecte correctamente meterRegistryUtils. Hazlo de una de estas dos formas (elige la más simple/estable según el código):
Opción A (rápida con reflection, sin levantar Spring):
Crea un SimpleMeterRegistry y un MeterRegistryUtils válido.
Inyéctalo al ObservationProcessor por reflection usando el nombre correcto del field: "meterRegistryUtils".
Código sugerido para el @Before (ajusta imports):
import io.micrometer.core.instrument.simple.SimpleMeterRegistry;
import org.springframework.test.util.ReflectionTestUtils;

@Before
public void setup() {
    // ... cualquier setup previo que ya haya

    SimpleMeterRegistry registry = new SimpleMeterRegistry();

    MeterRegistryUtils utils;
    try {
        // Si existe un constructor que recibe MeterRegistry:
        utils = new MeterRegistryUtils(registry);
    } catch (Throwable t) {
        // Si no existe, créalo vacío y setea su campo interno 'meterRegistry'
        utils = new MeterRegistryUtils();
        ReflectionTestUtils.setField(utils, "meterRegistry", registry);
    }

    // Inyecta en el processor el field correcto:
    ReflectionTestUtils.setField(observationProcessor, "meterRegistryUtils", utils);
}
Importante: elimina cualquier línea anterior que haga setField(..., "meterRegistry", ...) porque ese field no existe.
Opción B (más limpio con Spring y cucumber-spring legacy):
Usa contexto Spring en los steps y mockea los beans; así evitamos reflection.
import cucumber.api.spring.CucumberContextConfiguration;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.beans.factory.annotation.Autowired;

@CucumberContextConfiguration
@SpringBootTest(classes = ObservationApplication.class)
public class ObservationServiceCucumberTestSteps {

    @Autowired
    private ObservationProcessor observationProcessor;

    @MockBean
    private MeterRegistryUtils meterRegistryUtils;

    @MockBean
    private io.micrometer.core.instrument.MeterRegistry meterRegistry;

    @Before
    public void setup() {
        // Si hace falta, stubear métodos de meterRegistryUtils con Mockito.when(...)
        // No usar ReflectionTestUtils aquí.
    }
}
Restricciones
No modificar pom.xml (hay mezcla legacy info.cukes y moderna io.cucumber, pero estamos ejecutando con cucumber.api.cli.Main y funciona).
Mantener Java 11.
Criterios de aceptación
El hook Before ya no lanza IllegalArgumentException por meterRegistry.
Los escenarios en src/test/resources/feature/ObservationRequestKafka.feature inician correctamente.
El reporte target/cucumber-report/index.html ya no muestra el error “Could not find field 'meterRegistry'…”.
La ejecución se hace con la Run Config existente (legacy CLI) o desde mvn test si hay runner.
Si ves que MeterRegistryUtils depende internamente de un MeterRegistry y no hay constructor adecuado, aplica el plan de fallback del snippet (crear MeterRegistryUtils() y setear su campo interno meterRegistry con SimpleMeterRegistry).
