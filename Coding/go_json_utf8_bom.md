# Go 反序列化 utf8-bom 的 json 文件出错

刚开始接触 go，用它写个小工具，配置文件是个 json 文件，使用 `json.Unmarshal` 来反序列化它，error 提示说 `“invalid character 'ï' looking for beginning of value” from json.Unmarshal”`。

原来是因为 json 文件的保存编码是 `utf8 bom`，这个格式 go 不支持，但 windows 下比较常见。解决方案是将 json 文件重新保存成不带 bom 的 `utf8`，也可以直接替换掉文档前面的 bom 信息，参考 [Got error "invalid character 'ï' looking for beginning of value” from json.Unmarshal](https://stackoverflow.com/questions/31398044/got-error-invalid-character-%c3%af-looking-for-beginning-of-value-from-json-unmar)