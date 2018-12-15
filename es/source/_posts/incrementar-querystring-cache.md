title: Incrementar el tamaño del querystring cache
date: 2018/12/15
thumbnail: /images/querystring-cache_th.jpg
categories:
- Sitecore
tags:
- logs
- cache
- patch

---
<img class="hero-img" src="/images/querystring-cache.jpg" alt="Querystring cache">
---
Como incrementar el tamaño del querystring cache con parcheo de configuración.
<!-- more -->

*Versiones utilizadas: Sitecore Experience Platform 9.0 rev. 171219 (9.0 Update-1).*

Estaba leyendo los historiales de log de sitecore en una implementación y encontré un mensaje de advertencia (WARN) repetidamente. Usualmente estos mensajes de advertencia son ignorados o causados por otras razones que deben ser investigadas. En este caso el mensaje decía que el `WebUtil.QueryStringCache` estaba siendo limpiado:

___WARN  WebUtil.QueryStringCache cache is cleared by Sitecore.Caching.Generics.Cache`1+DefaultScavengeStrategy[[System.String, mscorlib, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089]] strategy. Cache running size was 17.7 KB___

Mi primer pensamiento fue que iba a encontrar esta setting en un archivo de configuración y podría cambiarlo por un número más grande y adecuado para la implementación en la que trabajaba. Pero... esta setting no existe en ningún archivo. Cuando el sitio inicia, el tamaño de la cache aparece y dice:

_INFO  Cache created: 'WebUtil.QueryStringCache' (max size: 19KB, running total: 5596MB)_

Como no pude encontrar donde cambiar esta setting, abrí el ILSpy y comencé a investigar de dónde sitecore leía este 19KB. Encontré `Sitecore.Caching.CacheManager.GetNamedInstance("WebUtil.QueryStringCache", 20000L, true)` en `WebUtil.cs`, pero no pude descifrar como cambiar este valor. Le comenté a mi colega [Viacheslav](https://community.sitecore.net/members/viacheslavsedletskiy_5f00_1037964896) y él también se puso a investigar en las librerías de Sitecore; encontró que esta se puede cambiar por vía de un `cacheContainer` nombrado.

El parche para incrementar el `WebUtil.QueryStringCache` es el siguiente:
``` xml
<sitecore>
  <cacheContainerConfiguration>
    <cacheContainer name="WebUtil.QueryStringCache" type="Sitecore.Caching.Cache, Sitecore.Kernel" singleInstance="true">
      <param desc="name">WebUtil.QueryStringCache</param>
        <param desc="cacheSize">3500KB</param>
    </cacheContainer>
  </cacheContainerConfiguration>
</sitecore>
```
Para confirmar que el nuevo valor esta siendo utilizado, puedes leerlo cuando el sitio inicializa. El mensaje aparece como:
    
__INFO  Cache created: 'WebUtil.QueryStringCache' (max size: 3MB, running total: 5280MB)__

PD: Como no hay ninguna documentación oficial sobre este cambio, contacté al sitecore support para confirmar que este método es válido y me dijeron que si. Aquí una transcripción de la respuesta: 

> Esta cache no puede ser cambiada vía archivo de configuración. En la mayoría de los casos esta cache no debería ser modificada y la notificación debe ser ignorada... La mejor práctica para tunear las caches de Sitecore es vía las settings de cache max size pero como esta setting no existe para QueryStringCache, este método es aceptable... Hemos creado un ticket para que esta setting sea añadida, el numero es #220607.

---

Déjame saber qué piensas y/o si encuentras algún error.
*/adiós*
