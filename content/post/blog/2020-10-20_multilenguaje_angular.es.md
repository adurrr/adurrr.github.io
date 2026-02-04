+++
author = "Adur"
title = "Multilenguaje en Angular 10/9/8 con la librería ngx-translate (i18n)"
date = "2020-10-20"
description = "Tutorial de cómo añadir la funcionalidad de multilenguaje con i18n en un proyector de Angular"
featured = true
tags = [
    "angular",
    "multilenguaje",
    "programación",
    "typescript"
]
categories = [
    "desarrollo software",
]
series = ["Programación"]
aliases = [""]
thumbnail = "images/i18n.png"
+++

En esta entrada se explicarán los pasos a seguir para añadir la funcionalidad de multilenguaje con i18n en un proyector de Angular 10/9/8. Para este tutorial es necesario tener un proyecto que esté creado previamente.

## Añadir ngx-translate en Angular 10/9/8 
Se instalan los paquetes de `ngx-translate` con:
```bash
npm i @ngx-translate/core @ngx-translate/http-loader --save
```
* `@ngx-translate/core` incluye los servicios esenciales, la canalización y las directivas para convertir el contenido en varios idiomas.

* `@ngx-translate/http-loader` ayuda a obtener los archivos de traducción de un servidor web.

Se añaden los módulos en el archivo **app.module.ts**:
```typescript
import { TranslateLoader, TranslateModule } from '@ngx-translate/core';
import { TranslateHttpLoader } from '@ngx-translate/http-loader';

...

@NgModule({
  ...
    imports: [
      ...
      TranslateModule.forRoot({
        loader: {
          provide: TranslateLoader,
          useFactory: httpTranslateLoader,
          deps: [HttpClient]
        }
      })
    ],
    ...
})

export function httpTranslateLoader(http: HttpClient) {
  return new TranslateHttpLoader(http);
}

```

## Configuración de los archivos traducidos
Se abre la carpeta `assets` y se crea el directorio `i18n`. Dentro de ella se añadirán tantos ficheros **<codigo_idioma_país>.json** como idiomas distintos se quieran traducir. Por ejemplo el fichero **src/assets/es.json** contendrá lo siguiente:
```json
{
  "HOME":"Inicio",
  "ABOUT": "Acerca de",
  "SING_IN":"Iniciar sesión",
  "LOG_OUT": "Cerrar sesión"
}
```
Otro ejemplo puede ser el fichero en inglés **src/assets/en.json**: 
```json
{
  "HOME":"Home",
  "ABOUT": "About",
  "SING_IN":"Sing in",
  "LOG_OUT": "Log out"
}
```

## Implementar las traducciones con TranslateService
Importamos el servicio **TranslateService** en `app.component.ts`:
```typescript
import { TranslateService } from '@ngx-translate/core';
``` 
Inyectamos el servicio **TranslateService** en el constructor para tener acceso al servicio:
```typescript
export class AppComponent {
  constructor(
    public translate: TranslateService
  ) {
    translate.addLangs(['en', 'en']);
    translate.setDefaultLang('es');
  }
}
```

## Función elegir el idioma

Mediante la función **switchLang** se elegirá el idioma que queramos. Para ello es necesario añadirla en el archivo TypeScript del componente donde queramos que aparezca:

```typescript
switchLang(lang: string) {
  this.translate.use(lang);
}
```
Añadimos un desplegable para que se pueda elegir el idioma:
```html
<span class="form-inline">
  <select 
      class="form-control" 
      #selectedLang 
      (change)="switchLang(selectedLang.value)">
    <option *ngFor="let language of translate.getLangs()" 
      [value]="language"
      [selected]="language === translate.currentLang">
      {{ language }}
    </option>
  </select>
</span>
```

## Configuración de las traducciones con TranslatePipe
Por último, una vez definida la constante en los archivos de idiomas (**src/ssets/<codigo_idioma_país>.json**), mediante la función pipe `{{'HOME' | translate }}` se traducirá la constante **HOME** al idioma elegido. Por ejemplo:

```html
<nav class="navbar navbar-dark bg-primary">
  <div class="container">
    <a class="navbar-item">
      {{'HOME' | translate }}
    </a>
    <a class="navbar-item">
      {{'ABOUT' | translate }}
    </a>
    <span class="form-inline">
      <select class="form-control" #selectedLang (change)="switchLang(selectedLang.value)">
        <option *ngFor="let language of translate.getLangs()" [value]="language"
          [selected]="language === translate.currentLang">
          {{ language }}
        </option>
      </select>
    </span>
  </div>
</nav>
```
## Conclusión
Finalmente, se ha implementado el multilenguage de la apliación Angular usando `ngx-translate`. Se ha analizado cómo configurar un entorno para traducir una aplicación Angular, cómo acceder a los métodos del servicio de traducción y cómo usar TranslatePipe.

##### Enlaces consultados
[PositronX.io - Create Angular 10/9/8 MultiLingual Site with ngx-translate i18n Library](https://www.positronx.io/angular-internationalization-i18n-with-ngx-translate-tutorial/)