## 欢迎你，小猪皮～

> 这里是猪皮恶霸的知识库

​	美好的一天就从这里开始，面对那昂贵的服务器，用了一年又一年实在是顶不住啦最终选择在这里搭建一个免费的知识库当作是我的博客。

​	如果某一天，我能够获得永久免费的服务器那我会毫不犹豫的部署上ky-blog。

```java
    @Override
    public BasePageVO<DeviceRechargeVO> queryRechargeInfo(Long pageNo, Long pageSize, String deviceId) {
        log.info("GeneralDeviceMeterWebBizServiceImpl::queryRechargeInfo##deviceId:{}", deviceId);
        String subjectCode = getUserInfo().getSubjectCode();
        IPage<DeviceChargeRecordDO> list =
            deviceChargeRecordDao.getPageByDevId(pageNo, pageSize, deviceId, GeneralEnum.SCOPE_OF_BLUETOOTHMETER.getDesc(), subjectCode);
        log.info("GeneralDeviceMeterWebBizServiceImpl::queryRechargeInfo##list:{}", JSON.toJSONString(list));
        //数据整合
        List<DeviceRechargeVO> result = list.getRecords().stream().map(record ->
            DeviceRechargeVO.builder()
                .rechargeTime(record.getGmtPay())
                .rechargeAccount(record.getUserName())
                .rechargeAmount(record.getAmountTrade().toString())
                .pricesOrRate(record.getPrice().equals(new BigDecimal("0")) ? "" : record.getPrice().toString())
                .build())
            .collect(Collectors.toList());
        return new BasePageVO<DeviceRechargeVO>(list.getTotal(), pageNo, pageSize, result);
    }
```

