---
title: "추상 팩토리 패턴과 전략패턴 활용"
description:
date: 2024-08-04
update: 2021-08-04
tags:
  - Linux
---
데이터베이스 모니터링 아키텍처를 설계 중 여러 데이터베이스의 Collector, Loader를 사용하기 위해 추상 팩토리 패턴, 전략패턴을 활용하여
아키텍처를 설계해보았습니다.
## 추상 팩토리 패턴
객체 생성 패턴 중 하나로, 관련된 객체들을 하나의 팩토리(Factory)에서 생성하도록 하는 디자인 패턴입니다.객체 생성에 대한 구체적인 클래스를 감추고, 상위 인터페이스만을 통해 객체를 생성함으로써 코드의 유연성과 확장성을 높이는 데 도움이 됩니다.
### 추상 팩토리
``` java
public interface MonitoringFactory {
  Collector createCollector();
  Loader createLoader();
}

public class OracleMonitoringFactory implements MonitoringFactory {
  @Override
  public Collector createCollector() {
    return new OracleCollector();
  }

  @Override
  public Loader createLoader() {
    return new OracleLoader();
  }
}

public class MysqlMonitoringFactory implements MonitoringFactory {
  @Override
  public Collector createCollector() {
    return new MysqlCollector();
  }

  @Override
  public Loader createLoader() {
    return new MysqlLoader();
  }
}

public interface Collector<T> {
  List<T> collect();
}

public interface Loader<T> {
  void load(List<T> entities);
}
```
## 전략패턴
전략패턴이란 객체의 행위를 클래스로 캡슐화하여 행위를 바꾸는 패턴입니다.
특정 작업을 수행하는 방법이 여러 가지일 때, 유용합니다.
``` java
public abstract class MonitoringStrategy {
  private MonitoringFactory monitoringFactory;

  void monitoring() {
    Collector collector = monitoringFactory.createCollector();
    Loader loader = monitoringFactory.createLoader();
    List<T> entities = collector.collect();
    loader.load(entities);
  }

  public MonitoringStrategy(MonitoringFactory monitoringFactory) {
    this.monitoringFactory = monitoringFactory;
  }

}

public class OracleMonitoringStrategy extends MonitoringStrategy {
  private MonitoringFactory monitoringFactory;

  public OracleMonitoringStrategy(MonitoringFactory monitoringFactory) {
    super(monitoringFactory);
  }
}

public class MysqlMonitoringStrategy extends MonitoringStrategy {
  private MonitoringFactory monitoringFactory;

  public MysqlMonitoringStrategy(MonitoringFactory monitoringFactory) {
    super(monitoringFactory);
  }
}

public class DatabaseVendorMonitoringStrategy {
  private HashMap<DatabaseVedor, MonitoringStrategy> databaseVendorMonitoringStrategy = new HashMap();
  
  static {
    databaseVendorMonitoringStrategy.put(DatabaseVendor.Oracle, new OracleMonitoringStrategy());
    databaseVendorMonitoringStrategy.put(DatabaseVendor.Mysql, new MysqlMonitoringStrategy());
  }
  
  public void monitoring(DatabaseVendor databaseVendor) {
    MonitoringStrategy monitoringStrategy = getMonitoringStrategyBy(databaseVendor);
    monitoringStrategy.monitoring();
  }

  private getMonitoringStrategyBy(DatabaseVendor databaseVendor) {
    if(!databaseVendorMonitoringStrategy.containsKey(databaseVendor)) {
      throw new RuntimeException("Database Vendor is not supported" + databaseVendor);
    }
    return databaseVendorMonitoringStrategy.get(databaseVendor);
  }
}
```

