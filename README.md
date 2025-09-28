You are my coding copilot for a Spring Boot 2.7.18 / Java 11 project that transforms Observation messages to FHIR R4 using JOLT. The module is “fhir-hl-observation-transformer”.

GOAL
Add two new Cucumber scenarios (vital-signs + body-height) and fix undefined steps so all tests pass green.

CONTEXT
- Feature file: src/test/resources/ObservationRequestKafka.feature
- Step defs class: src/test/java/**/cucumber/ObservationServiceCucumberTestSteps.java
- Existing flow (reuse it): producer → listener → doCustomProcessing → doJoltTransformation → parseEncodeToJsonString → FHIR JSON string in `fhirFormat`
- Existing expected JSONs: 
  - src/test/resources/ObservationBodyHeightExpected.json (present)
  - src/test/resources/ObservationBMIExpected.json (present)

WHAT TO DO
1) Update ObservationRequestKafka.feature: append two scenarios that REUSE the existing step wording as much as possible:
   - Scenario A (vital-signs, Heart Rate):
     - Use: 
       When kafka producer sends the entity message to the ENTITY_TOPIC from file "src/test/resources/ObservationInputHeartRate.json"
       Then kafka Listener listens the entity message from the ENTITY_TOPIC
       And client calls doCustomProcessing with the message
       And client calls doJoltTransformation with the message
       And receives the joltString as Response
       And client calls parseEncodeToJsonString with the joltString
       And receives the FhirFormat as Response
       And FHIR output should equal to file "src/test/resources/ObservationHeartRateExpected.json"
   - Scenario B (Body Height): same steps, input "ObservationInputBodyHeight.json" and expected "ObservationBodyHeightExpected.json"

2) Implement ANY missing step definitions to remove “Undefined step reference”:
   - @When("kafka producer sends the entity message to the ENTITY_TOPIC from file {string}")
     -> Read file into `jsonRawData` (or the variable the pipeline expects) to drive the rest of the flow.
   - @Then("FHIR output should equal to file {string}")
     -> Read expected JSON and compare with `fhirFormat` using JSONAssert NON_EXTENSIBLE.
   - If feature still uses "validate the response", implement:
     @Then("validate the response") { assert against an expected file OR remove usage if redundant. }

3) Create test data files:
   - src/test/resources/ObservationInputHeartRate.json  (simple HR payload used by current JOLT)
   - src/test/resources/ObservationHeartRateExpected.json (FHIR Observation with category vital-signs, code LOINC 8867-4)
   - src/test/resources/ObservationInputBodyHeight.json (if missing)

4) Keep wording EXACTLY as in the feature file so step matching succeeds.

5) Run tests and fix any remaining failures.

CODE SKETCHES (add in ObservationServiceCucumberTestSteps.java):
- Utility to read file:
  private static String readFile(String path) { ... FileInputStream + StreamUtils.copyToString(..., UTF_8) ... }

- From-file step:
  @When("kafka producer sends the entity message to the ENTITY_TOPIC from file {string}")
  public void sendMessageFromFile(String path) { this.jsonRawData = readFile(path); /* wire into existing flow as current tests do */ }

- Assert step:
  @Then("FHIR output should equal to file {string}")
  public void fhirOutputEqualsFile(String expectedPath) {
      String expected = readFile(expectedPath);
      org.skyscreamer.jsonassert.JSONAssert.assertEquals(expected, this.fhirFormat, org.skyscreamer.jsonassert.JSONCompareMode.NON_EXTENSIBLE);
  }

- Optional:
  @Then("validate the response")
  public void validateTheResponse() { /* delegate to fhirOutputEqualsFile with BMI/Height expected if needed */ }

ACCEPTANCE CRITERIA
- Both new scenarios execute end-to-end and pass.
- No “Undefined step reference”.
- Vital-signs scenario asserts category = "vital-signs" and uses LOINC 8867-4 for Heart Rate in expected JSON.
- `mvn -q test` is green.

AFTER YOU’RE DONE
Generate a concise commit message:
test(cucumber): add vital-signs (HR) and body-height scenarios; fix undefined steps
- New scenarios in ObservationRequestKafka.feature
- Param steps to load inputs and assert FHIR output (JSONAssert)
- Added/updated input/expected JSON files

Now, make the edits and show me the diff preview.