
# Recomendaciones de código limpio.

## Recomendaciones para eliminar if anidados.

El siguiente es un ejemplo de un caso tipico en el cual se presentan multiples if anidados.
Además de ello se evidencia declaraciones de try-catch y loops anidados.
```Java
@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit)
{
  List<AccruementDTO> accruementDTOs;
  AccruementResultEntity accruementResultEntity;
  BigDecimal totalInterests = new BigDecimal(0.0d);

  try {
    accruementDTOs = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());
    if(accruementDTOs != null && accruementDTOs.size() > 0) {
      for(AccruementDTO accruementDTO: accruementDTOs) {
        if(
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.NORMAL.getName()) ||
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.RISK.getName()) ||
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.DEFAULTER.getName())
        ) {
          accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
          if(accruementResultEntity != null) {
            if(debitCredit != null) {
              if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
                totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
              else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
                totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
            }
          }
        }
      }
    }
  } catch (AccruementException accruementException) {
    logger.error(accruementException);
  }

  return totalInterests;
}

```


### 1. Las validaciones se deben hacer antes de cualquier cosa.

El objetivo será, que si se debe validar algo , se haga en primera instancia, por ejemplo las validaciones de listas nulas o vacias. El siguiente es la refactorización aplicando esta recomendación.

```Java
@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {

  BigDecimal totalInterests = new BigDecimal(0.0d);

  List<AccruementDTO> accruementDTOs;
  AccruementResultEntity accruementResultEntity;
  try {
    accruementDTOs = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());

    if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
    }

      // Aquí se elimina un if Anidado.
    for(AccruementDTO accruementDTO: accruementDTOs) {
        if(
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.NORMAL.getName()) ||
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.RISK.getName()) ||
            accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.DEFAULTER.getName())
        ) {
          accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
          if(accruementResultEntity != null) {
            if(debitCredit != null) {
              if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
                totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
              else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
                totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
            }
          }
        }
    }

  } catch (AccruementException accruementException) {
    logger.error(accruementException);
  }

  return totalInterests;
}
```

### 2. Las validaciones dentro de if deben ser claros y dicientes.

Es muy comun que se presenten validaciones que tienden a ser complejas y se embeban dentro de una declaración if,
por lo cual se hace poco legible la validación que se intenta a realizar, por lo cual se sugiere crear un método
aparte que permita realizar dicho proceso de forma más clara. Veamos el segmento de código que se presenta en el loop.

```java
if(
    accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.NORMAL.getName()) ||
    accruementDTO.getName().equals(AccruementInterestPeriodEnumDTO.RISK.getName()) ||
    accruementDTO.getName().equals()
)
```

La anterior declaración se puede embeber en un nuevo método de la siguiente forma
```java
protected boolean isAccruementInterestValid(String accruementName) {
  // Aqui no se hace validación de si el accruementName es nulo ya que nunca será igual a
  // los valores contenidos en la lista.
  return Arrays.asList(new String[] {
    AccruementInterestPeriodEnumDTO.NORMAL.getName(),
    AccruementInterestPeriodEnumDTO.RISK.getName(),
    AccruementInterestPeriodEnumDTO.DEFAULTER.getName()
  }).contains(accruementName);
}

```
Posteriormente la validación dentro del loop se vería de la siguiente forma:


```java
for(AccruementDTO accruementDTO: accruementDTOs) {
    // refactorización de código, mucho más legible y mantenible.
    if(this.isAccruementInterestValid(accruementDTO.getName())) {
      accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
      if(accruementResultEntity != null) {
        if(debitCredit != null) {
          if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
          else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
        }
      }
    }
}
```

Antes de continuar, debemos refactorizar de nuevo nuestro código aplicando la recomendación 1. En esta ocasión la realizaremos dentro del loop.  De tal forma que si no cumple con un accruement valido procese el siguiente registro. Definido lo anterior el código quedaría de la siguiente forma.

```java
for(AccruementDTO accruementDTO: accruementDTOs) {
    // Regla 2. Una funcion embebida  legible y mantenible.
    // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
    if(!this.isAccruementInterestValid(accruementDTO.getName())) {
      // Como no es valido el accruement name, entonces debe pasar al siguiente
      // registro a ser procesado.
      continue;
    }
      accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
      if(accruementResultEntity != null) {
        if(debitCredit != null) {
          if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
          else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
        }
      }
    }
}
```

### 3. ¿Por qué hacer multiples if cuando puedes hacer solo uno?

Se aprecia que al consultar un resultado de accruement se deben realizar nuevas validaciones, actualmente e encuentra de la siguiente forma:

```java
accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
if(accruementResultEntity != null) {
  if(debitCredit != null) {
    ...
  }
}
```

Se puede abreviar y hacer un poco más legible para el desarrollador de la siguiente frozenReimbursingBank

```java
accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
if(accruementResultEntity != null && debitCredit != null) {
  ....
}
```

De esta forma se ha eliminado un if. Pero un momento cada que se presente un proceso  a relizar después de una validación se debe aplicar el principio 1. Las validaciones se deben hacer antes que cualquier cosa.


```java
@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {

  BigDecimal totalInterests = new BigDecimal(0.0d);

  List<AccruementDTO> accruementDTOs;
  AccruementResultEntity accruementResultEntity;
  try {
    accruementDTOs = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());

    // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
    if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
    }

    for(AccruementDTO accruementDTO: accruementDTOs) {
          // Regla 2. Una funcion embebida  legible y mantenible.
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(!this.isAccruementInterestValid(accruementDTO.getName())) {
            // Como no es valido el accruement name, entonces debe pasar al siguiente
            // registro a ser procesado.
            continue;
          }
          accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());

          // Regla 3. ¿Por qué hacer multiples if cuando puedes hacer solo uno?
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(accruementResultEntity == null || debitCredit == null) {
            continue;
          }

          if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
          else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
            totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
          }
    }

  } catch (AccruementException accruementException) {
    logger.error(accruementException);
  }

  return totalInterests;
}
```

### 4. Valores de retorno por default en funciones.

Cuando se tienen funciones que retornan valores y nos encontramos aplicando el principio 1, de tal forma que  si tienes un if que no cumple con una validación siempre debe existir una respuesta por default. En la funcion `calculateTotalInterestAmount` se observa que retorna un `BigDecimal`. Al proponer valores por default refactorizando
se puede observar que queda de la siguiente forma.

```Java
@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {

    BigDecimal totalInterests = new BigDecimal(0.0d);

    if (applicationSanchileDTO == null || (debitCredit == null || (debitCredit != null && debitCredit.isEmpty())) ) {
      return totalInterests;
    }

    List<AccruementDTO> accruementDTOs = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());


    if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
    }

   ....
}
```

Aplicando el principio 2, podemos refactorizar un poco más y crear una función que valide los parámetros de entrada de la función.

```Java

protected boolean areValidParameters(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit ) {
  return applicationSanchileDTO == null || (debitCredit == null || (debitCredit != null && debitCredit.isEmpty()));
}
```


```Java
@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {

    BigDecimal totalInterests = new BigDecimal(0.0d);
    if (this.areValidParameters(applicationSanchileDTO,debitCredit) ) {
      return totalInterests;
    }

    List<AccruementDTO> accruementDTOs = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());


    if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
    }

   ....
}
```


## Funciones con responsabilidad única

Anteriormente se creo una validacion `isAccruementInterestValid` con el objetivo de segmentar bloques de código, que se encargaran de hacer una sola cosa y que la haga bien, en este caso se validaba si era un accruement valido a procesar a partir del nombre.

Vamos ahora a enfocarnos en el proceso real que se encuentra dentro del loop de la función `calculateTotalInterestAmount`,
Si recordamos dejamos el siguiente segmento de código

``` Java
if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null)
   totalInterests = totalInterests.add(accruementResultEntity.getCounterValueDebit());
 else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null)
   totalInterests = totalInterests.add(accruementResultEntity.getCounterValueCredit());
 }
```
Este segmento podría crecer de  multiples formas que no tenemos certeza ya que a futuro los requerimientos pueden cambiar, por lo cual se podría agregar multiples reglas para hacer dicho calculo.  Lo mejor será crear un función que me permita hacer este proceso de forma ceparada con una responsabilidad única, calcular el interés de un resultado de accruement.


``` Java
protected BigDecimal  calculateInteres(String debitCredit,AccruementResultEntity accruementResultEntity accruementResultEntity) {
   if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null) {
      return accruementResultEntity.getCounterValueDebit();
   } else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null) {
     return accruementResultEntity.getCounterValueCredit();
   }
}

```

Supongamos que el requerimiento cambio  y que en el calculo del interés se agrega otro valor y otra validación.


``` Java
protected BigDecimal  calculateInteres(String debitCredit,AccruementResultEntity accruementResultEntity accruementResultEntity, BigDecimal anotherInteres,Date lastDate ) {
   Date today = new Date();
   if (lastDate.isAfter(today)) {
     return anotherInteres;
   }
   if(debitCredit.equals(DEBIT) && accruementResultEntity.getCounterValueDebit() != null) {
      return accruementResultEntity.getCounterValueDebit();
   } else if(debitCredit.equals(CREDIT) && accruementResultEntity.getCounterValueCredit() != null) {
     return accruementResultEntity.getCounterValueCredit();
   }
}

```
Como podemos ver esta funcuón no afecta el bloque principal del loop si no de del calculo del interés en sí.

## Los try-catch deberían tener valores por default a retornar.

Los segmentos de bloque `try-catch`  se crearón para controlar comportamientos no esperados, como por ejemplo la conexión a una base de datos, es un factor externo. La conexión a colas tipo MQ. Pero hay excepciones que podemos controlar y retornar un valor por default. Aquí hay que tener en cuenta si se debe controlar la excepción o debe ser propagada.

Veamos un ejemplo en el que debería ser controlada. Volvamos a nuestra función refactorizada.
```java

@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {
  BigDecimal totalInterests = new BigDecimal(0.0d);

  // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
  if (this.areValidParameters(applicationSanchileDTO,debitCredit) ) {
    return totalInterests;
  }

  try {  
    List<AccruementDTO> accruementDTOss = accruementService.getAccruementsByTransactionId(applicationSanchileDTO.getTransactionId());

    // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
    if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
    }

    for(AccruementDTO accruementDTO: accruementDTOs) {
          // Regla 2. Una funcion embebida  legible y mantenible.
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(!this.isAccruementInterestValid(accruementDTO.getName())) {
            // Como no es valido el accruement name, entonces debe pasar al siguiente
            // registro a ser procesado.
            continue;
          }
          AccruementResultEntity accruementResultEntity = accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());

          // Regla 3. ¿Por qué hacer multiples if cuando puedes hacer solo uno?
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(accruementResultEntity == null || debitCredit == null) {
            continue;
          }

          BigDecimal interes = this.calculateInteres(debitCredit, accruementResultEntity);
          totalInterests.add(interes);
    }

  } catch (AccruementException accruementException) {
    logger.error(accruementException);
  }

  return totalInterests;
}
```

Se aprecia un bloque en el cuál hay un `try-catch` este se presenta básicamente por dos llamdos

```java
1. accruementService.getAccruementsByTransactionId(...)
2. accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(...);  

```

En caso se debe analizar
1. ¿Qué pasa si por error quedo mal programado el query en el repositorio?
2. ¿Qué pasa si no encuentra registros?

Bueno como este es un ejemplo en el que se deben controlar las "Excepciones" lo que se debe hacer es crear funciones con responsabilidades únicas que se encarguen de retornar valores por default en el segmento `catch`


```java
protected List<AccruementDTO> getAccruements(String transactionId) {
  try {
    return accruementService.getAccruementsByTransactionId(transactionId);
  } catch(AccruementException e) {
    return null;
  }
}

protected AccruementResultEntity getAccruementResult(int accruementId) {
  try {
    return  accruementResultRepositorySanChileBack.findByLastProcessedDateByAccruementId(accruementDTO.getId());
  } catch(Exception e) {
    return null;
  }
}
```

En los dos métodos anteriores vemos que en el segmento `catch` se retorna `null`, ya que no otra información por default para retornar, pero previamente ya habíamos validados los valores nullos para esas consultas.

```java

@Transactional
private BigDecimal calculateTotalInterestAmount(LoansApplicationSanChileBackDTO applicationSanchileDTO, String debitCredit) {
  BigDecimal totalInterests = new BigDecimal(0.0d);

  // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
  if (this.areValidParameters(applicationSanchileDTO,debitCredit) ) {
    return totalInterests;
  }

  // Try and catch previamente controlados  
  List<AccruementDTO> accruementDTOss = this.getAccruements(applicationSanchileDTO.getTransactionId());

  // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
  if (accruementDTOs == null || (accruementDTOs != null && accruementDTOs.isEmpty()) ) {
      return totalInterests;
  }

  for(AccruementDTO accruementDTO: accruementDTOs) {
          // Regla 2. Una funcion embebida  legible y mantenible.
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(!this.isAccruementInterestValid(accruementDTO.getName())) {
            // Como no es valido el accruement name, entonces debe pasar al siguiente
            // registro a ser procesado.
            continue;
          }
          // try and catch previamente controlado.
          AccruementResultEntity accruementResultEntity = this.getAccruementResult(accruementDTO.getId());

          // Regla 3. ¿Por qué hacer multiples if cuando puedes hacer solo uno?
          // Regla 1. Validar primero todo antes de continuar con cualquier cosa.
          if(accruementResultEntity == null || debitCredit == null) {
            continue;
          }
          // Funciones con responsabilidad única.
          BigDecimal interes = this.calculateInteres(debitCredit, accruementResultEntity);
          totalInterests.add(interes);
  }

  return totalInterests;
}
```
