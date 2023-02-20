##实现
通过go "encoding/json" 将收到的json数据json.Marshal(data)到文件中，每次调用写入内容通过”，“隔开，最终文件内呈现一个数组对象
## 命令行调用格式化文件
```bash
curl -X POST "http://192.168.11.58:21805/sd_file/format"
```
## 命令行调用新增数据
```bash
curl -X POST "http://192.168.11.58:21805/sd_file/add" -H "Content-Type: application/json" -d '{"labels": {"mib": "if_mib","brand": "DP-WAFNFV","hostname": "pbr-dev-waf-backup","model": "zdyNB"},"targets": ["192.168.11.42"]}'
```
```json
[root@prometheus conf]# cat snmp_device.yml 
[{"labels":{"mib":"if_mib","brand":"DP-FW1000","hostname":"pbr-dev-ngfw-master","model":"zdyNB"},"targets":["192.168.11.36"]},{"labels":{"mib":"if_mib","brand":"DP-FW1000","hostname":"pbr-dev-ngfw-backup","model":"zdyNB"},"targets":["192.168.11.38"]},{"labels":{"mib":"if_mib","brand":"DP-WAFNFV","hostname":"pbr-dev-waf-backup","model":"zdyNB"},"targets":["192.168.11.40"]},{"labels":{"mib":"if_mib","brand":"DP-WAFNFV","hostname":"pbr-dev-waf-backup","model":"zdyNB"},"targets":["192.168.11.42"]}]
```
## 代码
```go
package main

import (
	"encoding/json"
	"fmt"
	"github.com/gin-gonic/gin"
	"io/ioutil"
	"os"
)

type Labels struct {
	Mib      string `json:"mib"`
	Brand    string `json:"brand"`
	Hostname string `json:"hostname"`
	Model    string `json:"model"`
}

type FileDsData struct {
	Labels  Labels   `json:"labels"`
	Targets []string `json:"targets"`
}

func main() {
	r := gin.Default()

	r.POST("/sd_file/add", AddSDFIle)
	r.POST("/sd_file/format", FormatJsonFile)

	r.Run(":21805")
}

func AddSDFIle(c *gin.Context) {
	var data FileDsData
	if err := c.ShouldBindJSON(&data); err != nil {
		c.JSON(400, gin.H{"error": err.Error()})
		return
	}

	// Read the existing data from the file
	file, err := os.OpenFile("/opt/monitor/prometheus/conf/snmp_device.yml", os.O_RDWR, 0644)
	if err != nil {
		// If the file does not exist, create it
		file, err = os.Create("/opt/monitor/prometheus/conf/snmp_device.yml")
		if err != nil {
			c.JSON(500, gin.H{"error": err.Error()})
			return
		}
	}
	defer file.Close()

	// Read the existing content of the file
	existingData, err := ioutil.ReadAll(file)
	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}

	// Write the new data to the file
	if len(existingData) > 0 {
		existingData = existingData[:len(existingData)-1]   // Remove the last comma
		existingData = append(existingData, []byte(",")...) // Add a comma
	} else {
		existingData = append(existingData, []byte("[")...) // Add an opening bracket
	}

	newData, err := json.Marshal(data)
	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}
	existingData = append(existingData, newData...)
	existingData = append(existingData, []byte("]")...) // Add a closing bracket

	// Write the new data back to the file
	_, err = file.WriteAt(existingData, 0)
	if err != nil {
		c.JSON(500, gin.H{"error": err.Error()})
		return
	}
	c.JSON(200, gin.H{"message": "success"})
}

func FormatJsonFile(c *gin.Context) {
	// 打开JSON文件
	file, err := os.OpenFile("/opt/monitor/prometheus/conf/snmp_device.yml", os.O_RDWR, 0644)
	if err != nil {
		uc := fmt.Sprintln(err)
		c.JSON(500, uc)
		return
	}
	defer file.Close()

	// 读取并解码JSON数据
	var data interface{}
	decoder := json.NewDecoder(file)
	err = decoder.Decode(&data)
	if err != nil {
		uc := fmt.Sprintln(err)
		c.JSON(500, uc)
		return
	}

	// 格式化JSON数据
	formatted, err := json.MarshalIndent(data, "", "    ")
	if err != nil {
		uc := fmt.Sprintln(err)
		c.JSON(500, uc)
		return
	}

	// 将格式化后的JSON数据写回文件中
	err = ioutil.WriteFile("/opt/monitor/prometheus/conf/snmp_device.yml", formatted, 0644)
	if err != nil {
		uc := fmt.Sprintln(err)
		c.JSON(500, uc)
		return
	}

	c.JSON(200, "message：格式化成功")
}
```
