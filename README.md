# JavaToKotlin
Java 코드에서 Kotlin으로 변형하는 공부

[Batch Code Java to Kotlin.md](https://github.com/HyunWoo9930/JavaToKotlin/files/11059206/Batch.Code.Java.to.Kotlin.md)

# Batch Code Java to Kotlin

#### Java Code

```java
package skt.ip;

import com.google.gson.JsonElement;
import com.google.gson.JsonObject;
import com.google.gson.JsonParser;
import java.io.BufferedReader;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.InputStreamReader;
import java.io.PrintStream;
import java.net.HttpURLConnection;
import java.net.URL;
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.SQLException;
import java.util.Map.Entry;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.io.output.TeeOutputStream;

@Slf4j
public class ApiScheduler {

  // TODO 2. 오류나, error 메세지 수정.
  // TODO 3. Batch API 만들기


  private static final String[] apis = {"acl", "address_table", "arp", "bfd", "bgp", "interfaces",
      "lldp", "mlag", "ntp", "ospf", "ospfv3", "platform", "redun_check", "routing", "varp",
      "version", "vrf", "vrrp", "vxlan"};

  //TestBed
//  private static final String url = "jdbc:postgresql://60.30.137.168:15002/tsdn"; // 데이터베이스 URL
//  private static final String username = "tsdn_atlas"; // 데이터베이스 사용자 이름
//  private static final String password = "atlastsdn"; // 데이터베이스 비밀번호

//   Local
  private static final String url = "jdbc:postgresql://localhost:5432/ipbh"; // 데이터베이스 URL
  private static final String username = "postgres"; // 데이터베이스 사용자 이름
  private static final String password = ""; // 데이터베이스 비밀번호

  public ApiScheduler() {
    for (String key : apis) {
      key = "cm_svc_" + key;
      String key_last = key + "_last";
      // 1. 원래 있던 Key_last를 지움.
      String sql = "DROP TABLE IF EXISTS " + key_last + ";";
      putSql(sql);
      // 2. 현재 key에 있는 값을 key_last로 복사.
      sql = "CREATE TABLE " + key_last + " AS "
          + "SELECT * "
          + "FROM " + key + ";";
      putSql(sql);
      // 3. key에 있는 데이터들을 delete
      sql = "DELETE FROM " + key + ";";
      putSql(sql);
      log.debug("{} data rebuild success", key);
    }
  }

//  TestBed
//  private static final List<String> adminIps = List.of(
//      "60.30.163.2", "60.30.163.3", "60.30.163.4", "60.30.163.5", "60.30.163.6", // Arista
//      "60.30.163.7", "60.30.163.8", "60.30.163.9", "60.30.163.10", "60.30.163.11",
//      "60.30.163.12", "60.30.163.13", "60.30.163.14", "60.30.163.15", "60.30.163.16",
//      "60.30.163.17", "60.30.163.18", "60.30.163.19", "60.30.163.20", "60.30.163.21",
//      "60.30.163.22", "60.30.163.23", "60.30.163.24", "60.30.163.33", "60.30.163.34",
//      "60.30.163.35", "60.30.163.36", "60.30.163.37", "60.30.163.38", "60.30.163.39",
//      "60.30.163.40", "60.30.163.41", "60.30.163.42", "60.30.163.43", "60.30.163.44",
//      "60.30.163.45", "60.30.163.46", "60.30.163.47", "60.30.163.48", "60.30.163.49",
//      "60.30.163.50", "60.30.163.51", "60.30.163.52", "60.30.163.62", "60.30.163.65",
//      "60.30.163.66", "60.30.163.67", "60.30.163.68", "60.30.163.69", "60.30.163.70",
//      "60.30.163.71", "60.30.163.72", "60.30.163.73", "60.30.163.74", "60.30.163.75",
//      "60.30.163.76", "60.30.163.86", "60.30.163.87",
//      "60.30.163.130", "60.30.163.131", "60.30.163.132", "60.30.163.133", "60.30.163.134", //Huawei
//      "60.30.163.135", "60.30.163.136", "60.30.163.137", "60.30.163.138", "60.30.163.140",
//      "60.30.163.141", "60.30.163.142", "60.30.163.143", "60.30.163.144", "60.30.163.145",
//      "60.30.163.148", "60.30.163.149", "60.30.163.150", "60.30.163.151", "60.30.163.152",
//      "60.30.163.153", "60.30.163.154", "60.30.163.155", "60.30.163.156", "60.30.163.157"
//  );
//  private static final String basicUrl = "http://localhost:8080/v1/od/svsw/";

  //Local

  private static final String[] adminIps = {"192.168.100.246", "192.168.100.235", "192.168.100.236"};
  private static final String basicUrl = "http://localhost:8899/v1/od/svsw/";

  public void startApiCallerWithThreadsAndScheduler(String api, String param) {
    for (String adminIp : adminIps) {
      String apiUrl = basicUrl + api + "/" + param + "?"
          + "adminIp=" + URLEncoder.encode(adminIp, StandardCharsets.UTF_8);

      ApiCallerTask apiCallerTask = new ApiCallerTask(apiUrl);
      apiCallerTask.run();
    }
  }

  private static class ApiCallerTask implements Runnable {

    private final String apiUrl;

    private ApiCallerTask(String apiUrl) {
      this.apiUrl = apiUrl;
    }
    @Override
    public void run() {
      try {
        URL url = new URL(apiUrl);
        log.debug("{} collecting start", url);
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("GET");
        conn.setRequestProperty("Content-Type", "application/json");
        conn.setConnectTimeout(60000);
        conn.setReadTimeout(60000);

        if (conn.getResponseCode() == HttpURLConnection.HTTP_OK) {
          BufferedReader in = new BufferedReader(new InputStreamReader(conn.getInputStream()));
          String inputLine;
          StringBuilder response = new StringBuilder();
          while ((inputLine = in.readLine()) != null) {
            response.append(inputLine);
          }
          in.close();

          JsonParser parser = new JsonParser();

          JsonObject jsonResponse = parser.parse(response.toString()).getAsJsonObject();

          String adminIp = jsonResponse.get("adminIp").toString();
          String hostname = jsonResponse.get("hostname").toString();

          log.debug("adminIp = {}", adminIp);
          log.debug("hostname = {}", hostname);

          jsonResponse.remove("adminIp");
          jsonResponse.remove("hostname");
          Entry<String, JsonElement> entry = jsonResponse.entrySet().iterator().next();
          String key = "cm_svc_" + entry.getKey();

          insertData(jsonResponse, adminIp, hostname, key);
        } else {
          log.debug("Failed to get response from server");
          log.debug("url = {}", apiUrl);
        }
      } catch (Exception e) {
        e.printStackTrace();
      }
    }
  }

  static void insertData(JsonObject jsonData,
      String adminIp, String hostname, String key) {
    log.debug("key = {}", key);

    String sql = "INSERT INTO " + key
        + " (adminip, hostname, data, collection_time, updated_at) VALUES (?, ?, cast(? as jsonb), NOW(), NOW())";

    try (Connection connection = DriverManager.getConnection(ApiScheduler.url,
        ApiScheduler.username, ApiScheduler.password);
        PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
      preparedStatement.setString(1, adminIp);
      preparedStatement.setString(2, hostname);
      preparedStatement.setString(3, jsonData.toString());
      preparedStatement.executeUpdate();
      System.out.println("Data inserted successfully.");
    } catch (SQLException e) {
      throw new RuntimeException(e);
    }
  }

  public static void main(String[] args) throws FileNotFoundException {
    // 파일 출력 스트림 생성
    PrintStream consoleOut = System.out;
    FileOutputStream fos = new FileOutputStream("log_file.log");

    // 출력 스트림(PrintStream)을 파일 출력 스트림(FileOutputStream)과 콘솔 출력 스트림(System.out)으로 변경
    System.setOut(new PrintStream(new TeeOutputStream(consoleOut, fos)));

    // 출력 스트림을 다시 원래대로 변경
    System.setOut(System.out);

    ApiScheduler apiCaller = new ApiScheduler();
    while (true) {
      for (String api : apis) {
        apiCaller.startApiCallerWithThreadsAndScheduler(api, "oper");
      }
    }
  }

  public static void putSql(String sql) {
    try (Connection connection = DriverManager.getConnection(url, username, password);
        PreparedStatement preparedStatement = connection.prepareStatement(sql)) {
      preparedStatement.executeUpdate();
    } catch (SQLException e) {
      throw new RuntimeException(e);
    }
  }
}


```



#### Kotlin

```kotlin
package skt.ip.svswapischedulerkotlin

import com.google.gson.JsonElement
import com.google.gson.JsonObject
import com.google.gson.JsonParser
import org.apache.commons.io.output.TeeOutputStream
import java.io.File
import java.io.PrintStream
import java.net.HttpURLConnection
import java.net.URL
import java.net.URLEncoder
import java.nio.charset.StandardCharsets
import java.sql.DriverManager

val logFile = File("log_file.log")
val teeOutputStream = TeeOutputStream(System.out, logFile.outputStream())

val apis = arrayOf("acl", "address_table", "arp", "bfd", "bgp", "interfaces",
        "lldp", "mlag", "ntp", "ospf", "ospfv3", "platform", "redun_check", "routing", "varp",
        "version", "vrf", "vrrp", "vxlan")

/* TestBed
val url : String = "jdbc:postgresql://60.30.137.168:15002/tsdn"
val username : String = "tsdn_atlas"
val password : String = "atlastsdn"
val adminIps = arrayOf(
        "60.30.163.2", "60.30.163.3", "60.30.163.4", "60.30.163.5", "60.30.163.6", // Arista
        "60.30.163.7", "60.30.163.8", "60.30.163.9", "60.30.163.10", "60.30.163.11",
        "60.30.163.12", "60.30.163.13", "60.30.163.14", "60.30.163.15", "60.30.163.16",
        "60.30.163.17", "60.30.163.18", "60.30.163.19", "60.30.163.20", "60.30.163.21",
        "60.30.163.22", "60.30.163.23", "60.30.163.24", "60.30.163.33", "60.30.163.34",
        "60.30.163.35", "60.30.163.36", "60.30.163.37", "60.30.163.38", "60.30.163.39",
        "60.30.163.40", "60.30.163.41", "60.30.163.42", "60.30.163.43", "60.30.163.44",
        "60.30.163.45", "60.30.163.46", "60.30.163.47", "60.30.163.48", "60.30.163.49",
        "60.30.163.50", "60.30.163.51", "60.30.163.52", "60.30.163.62", "60.30.163.65",
        "60.30.163.66", "60.30.163.67", "60.30.163.68", "60.30.163.69", "60.30.163.70",
        "60.30.163.71", "60.30.163.72", "60.30.163.73", "60.30.163.74", "60.30.163.75",
        "60.30.163.76", "60.30.163.86", "60.30.163.87",
        "60.30.163.130", "60.30.163.131", "60.30.163.132", "60.30.163.133", "60.30.163.134", //Huawei
        "60.30.163.135", "60.30.163.136", "60.30.163.137", "60.30.163.138", "60.30.163.140",
        "60.30.163.141", "60.30.163.142", "60.30.163.143", "60.30.163.144", "60.30.163.145",
        "60.30.163.148", "60.30.163.149", "60.30.163.150", "60.30.163.151", "60.30.163.152",
        "60.30.163.153", "60.30.163.154", "60.30.163.155", "60.30.163.156", "60.30.163.157")
val basicurl = "http://localhost:8080/v1/od/svsw/"
*/

// local
val url: String = "jdbc:postgresql://localhost:5432/ipbh"
val username: String = "postgres"
val password: String = ""
val adminIps = arrayOf("192.168.100.246", "192.168.100.235", "192.168.100.236")
val basicurl = "http://localhost:8899/v1/od/svsw/"

fun main() {
    init()
    System.setOut(PrintStream(teeOutputStream))
    while (true) {
        startCallerByApi(apis)
    }
}

class ApiCallerTask(private val apiUrl: String) : Runnable {
    override fun run() {
        try {
            val url = URL(apiUrl)
            println("$url collecting start")
            (url.openConnection() as? HttpURLConnection)?.apply {
                requestMethod = "GET"
                setRequestProperty("Content-Type", "application/json")
                setConnectTimeout(60000)
                setReadTimeout(60000)

                if (responseCode == HttpURLConnection.HTTP_OK) {
                    inputStream.bufferedReader().use {
                        val jsonResponse = JsonParser().parse(it.readText()).asJsonObject
                        val adminIp = jsonResponse["adminIp"].asString
                        val hostname = jsonResponse["hostname"].asString
                        println("adminIp = $adminIp")
                        println("hostname = $hostname")

                        jsonResponse.remove("adminIp")
                        jsonResponse.remove("hostname")
                        val entry: Map.Entry<String, JsonElement> = jsonResponse.entrySet().iterator().next()
                        val key = "cm_svc_${entry.key}"

                        insertData(jsonResponse, adminIp, hostname, key)
                    }
                } else {
                    println("Failed to get response from server")
                    println("url = $apiUrl")
                }
            }
        } catch (e: Exception) {
            e.printStackTrace()
        }
    }
}

fun insertData(jsonData: JsonObject, adminIp: String, hostname: String, key: String) {
    val sql = "INSERT INTO $key (adminip, hostname, data, collection_time, updated_at) " +
            "VALUES (?, ?, cast(? as jsonb), NOW(), NOW())"
    DriverManager.getConnection(url, username, password).use { connection ->
        connection.prepareStatement(sql).use { preparedStatement ->
            preparedStatement.setString(1, adminIp)
            preparedStatement.setString(2, hostname)
            preparedStatement.setString(3, jsonData.toString())
            preparedStatement.executeUpdate()
            println("Data inserted successfully")
        }
    }
}

fun startApiCallerByAdminIp(api: String, param: String) {
    adminIps.forEachIndexed {index, adminIp ->
        val apiUrl = "$basicurl$api/$param?adminIp=${URLEncoder.encode(adminIp, StandardCharsets.UTF_8)}"
        ApiCallerTask(apiUrl).run()
    }
}

fun init() {
    for (i: String in apis) {
        val key = "cm_svc_$i"
        val key_last = "${key}_last"

        val sql = """
        |DROP TABLE IF EXISTS $key_last;
        |CREATE TABLE $key_last AS SELECT * FROM $key;
        |DELETE FROM $key;
        |""".trimMargin()

        putSql(sql)
        println("$key data rebuild success")
    }
}

fun putSql(sql: String) {
    val connection = DriverManager.getConnection(url, username, password)
    val preparedStatement = connection.prepareStatement(sql)
    preparedStatement.executeUpdate()
}

fun startCallerByApi(apis: Array<String>) {
    apis.forEach { startApiCallerByAdminIp(it, "oper") }
}


```



##### Kotlin 이 Java 보다 좋은점

1. 더 간결하고 가독성이 좋습니다.
   - Kotlin은 Java에 비해 코드 양이 적습니다. 또한, Nullable 타입과 Elvis 연산자 등의 기능을 지원하여 코드를 더 간결하고 가독성이 좋게 작성할 수 있습니다.
2. 안전하고 유연합니다.
   - Kotlin은 NullSafe 방식으로 동작하기 때문에 NullPointerException을 방지할 수 있습니다. 또한, Java와의 상호 운용성을 보장하기 때문에 Java 코드와 Kotlin 코드를 함께 사용할 수 있습니다.
3. 함수형 프로그래밍을 지원합니다.
   - Kotlin은 함수형 프로그래밍을 지원하기 때문에, 람다식을 사용하여 코드를 간결하고 가독성 좋게 작성할 수 있습니다. 또한, 함수형 프로그래밍을 지원하므로 코드의 안정성과 확장성을 높일 수 있습니다.
4. 개발 생산성을 높일 수 있습니다.
   - Kotlin은 Android 개발에서 지원되므로 안드로이드 앱 개발에서 많은 이점을 가져다줍니다. 또한, IDE의 지원도 우수하여 개발 생산성을 높일 수 있습니다.

