Scenario: Transform HEART RATE message into FHIR Observation (vital-signs)
  When kafka producer sends the entity message to the ENTITY_TOPIC from file "src/test/resources/ObservationInputHeartRate.json"
  Then kafka Listener listens the entity message from the ENTITY_TOPIC
  And client calls doCustomProcessing with the message
  And client calls doJoltTransformation with the message
  And receives the joltString as Response
  And client calls parseEncodeToJsonString with the joltString
  And receives the FhirFormat as Response
  And FHIR output should equal to file "src/test/resources/ObservationHeartRateExpected.json"

Scenario: Transform BODY HEIGHT message into FHIR Observation
  When kafka producer sends the entity message to the ENTITY_TOPIC from file "src/test/resources/ObservationInputBodyHeight.json"
  Then kafka Listener listens the entity message from the ENTITY_TOPIC
  And client calls doCustomProcessing with the message
  And client calls doJoltTransformation with the message
  And receives the joltString as Response
  And client calls parseEncodeToJsonString with the joltString
  And receives the FhirFormat as Response
  And FHIR output should equal to file "src/test/resources/ObservationBodyHeightExpected.json"
