<template>
  <div></div>
</template>

<script>
/*! Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
*  SPDX-License-Identifier: MIT-0
*/

const ridesTopic = 'theme-park-rides'
const { fromCognitoIdentityPool } = require("@aws-sdk/credential-providers")
import { mqtt, iot } from "aws-iot-device-sdk-v2";

export default {
  name: 'IoT',
  data() {
    return {
      init: false
    }
  },
  mounted: async function () {
    // For the workshop, if this isn't in the config, the user has not
    // attemped this module yet, so hide the feature.

    if (this.$appConfig.iot.poolId === '') return
    const AWSConfiguration = this.$appConfig.iot

    console.log('IoT mounted')
    const _store = this.$store
    // const _t = this.$t

    const clientId = 'theme-park-client-' + (Math.floor((Math.random() * 100000) + 1))

    const credentials = await fromCognitoIdentityPool({
      identityPoolId: AWSConfiguration.poolId,
      clientConfig: {
        region: AWSConfiguration.region
      }
    });

    const creds = await credentials();
    console.log("credentials", creds)

    let aws_region = AWSConfiguration.region
    let aws_access_id = creds.accessKeyId
    let aws_secret_key = creds.secretAccessKey
    let aws_sts_token = creds.sessionToken


    let config = iot.AwsIotMqttConnectionConfigBuilder.new_with_websockets()
      .with_clean_session(true)
      .with_client_id(clientId)
      .with_endpoint(AWSConfiguration.host)
      .with_credentials(aws_region, aws_access_id, aws_secret_key, aws_sts_token)
      .with_keep_alive_seconds(30)
      .build();

    const client = new mqtt.MqttClient();
    let connection = client.new_connection(config);
    console.log("Connecting...");

    connection.on("connect", (session_present) => {
      console.log('mqttClient connected')
      connection.subscribe(ridesTopic, mqtt.QoS.AtLeastOnce, handleMessageReceived)
    });

    function handleMessageReceived(topic, payload, dup, qos, retain) {
      const decoder = new TextDecoder("utf8");
      let payloadEnvelope = decoder.decode(new Uint8Array(payload));
      payloadEnvelope = JSON.parse(payloadEnvelope.toString());
      console.log("IoT::onMessage:",topic )
      console.log("Message", payloadEnvelope);

      if (payloadEnvelope.type === 'alert') {
        _store.commit('setParkAlert', JSON.parse(payloadEnvelope.msg))
      }
      if (payloadEnvelope.type === 'photoProcessed') {
        console.log('Photo processed: ', payloadEnvelope.message.URL)
        _store.commit('addPhoto', payloadEnvelope.message.URL)
        _store.commit('setParkAlert', {
          title: 'Photo processed',
          type: 'info',
          confirmButtonText: 'OK'
        })
      }
      // ride updates use the ridesTopic
      if (payloadEnvelope.type === 'summary') {
        _store.commit('updateRideTimes', payloadEnvelope.msg)
      }

    }

    connection.on("error", (error) => {
      console.log("mqttClient error:", error);

    });
  }
}
</script>
