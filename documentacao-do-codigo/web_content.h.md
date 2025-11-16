# web\_content.h

{% tabs %}
{% tab title="C" %}
```c
/**
 * @file web_content.h
 * @brief Interface para obter conteúdo web estático (HTML, CSS, JS)
 */

#ifndef WEB_CONTENT_H
#define WEB_CONTENT_H

/**
 * @brief Retorna o conteúdo CSS da aplicação
 * @return String com o CSS
 */
const char* get_style_css(void);

/**
 * @brief Retorna o conteúdo JavaScript da aplicação
 * @return String com o JS
 */
const char* get_script_js(void);

/**
 * @brief Retorna o HTML da página principal
 * @return String com o HTML
 */
const char* get_home_html(void);

#endif /* WEB_CONTENT_H */

```
{% endtab %}
{% endtabs %}
