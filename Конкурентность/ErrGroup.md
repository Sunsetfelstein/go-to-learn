Удобный и популярно используемый инструмент в Go для работы с группой горутин, когда нужно:

- запускать несколько задач параллельно
- получать первую же ошибку (или все ошибки — есть варианты)
- корректно дожидаться завершения всех горутин
- иметь возможность отмены через context

Самый популярный пакет: golang.org/x/sync/errgroup

|Метод/Свойство|Что делает|Когда использовать|
|---|---|---|
|`g.Go(func() error)`|Запускает горутину|Основной способ добавления задачи|
|`g.Wait()`|Ждёт все горутины, возвращает первую ошибку|Почти всегда в конце|
|`errgroup.WithContext()`|Создаёт группу + контекст, который отменяется при первой ошибке|Самый рекомендуемый способ в 2025–2026|
|`g.SetLimit(n)`|Ограничивает максимальное кол-во одновременно работающих горутин|При работе с большим количеством задач|
|`g.TryGo()`|Пытается запустить, возвращает bool (удалось ли)|Когда лимит может быть исчерпан|
|`g.Counter()`|Текущее количество активных горутин (с Go 1.21+)|Для отладки/логов|

``` go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"golang.org/x/sync/errgroup"
)

// Проверяет, доступен ли URL (возвращает nil если 200-299 или 304)
func checkURL(ctx context.Context, client *http.Client, url string) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodHead, url, nil)
	if err != nil {
		return fmt.Errorf("не удалось создать запрос для %s: %w", url, err)
	}

	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("%s → ошибка соединения: %w", url, err)
	}
	defer resp.Body.Close()

	if resp.StatusCode >= 200 && resp.StatusCode < 300 || resp.StatusCode == 304 {
		return nil
	}

	return fmt.Errorf("%s → статус %d", url, resp.StatusCode)
}

func main() {
	// Настраиваем клиента один раз (лучше переиспользовать)
	client := &http.Client{
		Timeout: 8 * time.Second,
		// Можно добавить Transport с настройками соединений, прокси и т.д.
	}

	urls := []string{
		"https://google.com",
		"https://github.com",
		"https://httpstat.us/200",
		"https://httpstat.us/404",
		"https://httpstat.us/500",
		"https://www.rust-lang.org",
		"https://non-existing-domain-xyz1234567890.com",
	}

	// Создаём группу с контекстом для возможности ранней отмены
	g, ctx := errgroup.WithContext(context.Background())

	// Ограничиваем количество одновременных проверок (например, 6)
	g.SetLimit(6)

	for _, url := range urls {
		url := url // захватываем переменную
		g.Go(func() error {
			err := checkURL(ctx, client, url)
			if err != nil {
				// Можно здесь логировать, но не прерываем остальные
				fmt.Printf("  ✗ %s\n", err)
				return err // возвращаем ошибку → группа остановится после всех
			}
			fmt.Printf("  ✓ %s\n", url)
			return nil
		})
	}

	// Ждём завершения всех проверок
	fmt.Println("Проверка запущена...\n")
	err := g.Wait()

	if err != nil {
		fmt.Printf("\nРезультат: хотя бы один URL недоступен (первая ошибка: %v)\n", err)
	} else {
		fmt.Println("\nВсе URL доступны ✓")
	}
}
```