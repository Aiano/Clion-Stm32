# 通过openocd+daplink

## 报错

### Error connecting DP: cannot read IDR

> https://eagletrt.wiki/common/shared/openocd-errors/

接线错误，接成了烧录烧录器的SWD接口

config.cfg配置

```
# 选择下载器
interface cmsis-dap
transport select swd
# 选择板子
source [find target/stm32f1x.cfg]
adapter speed 10000
```

