# Exercise 02

![WorkflowResult](https://github.com/franzfilip/mc-fh-cicd/actions/workflows/main.yml/badge.svg)

[Github Link](https://github.com/franzfilip/mc-fh-cicd)

## Aufsetzen von Docker
Zuerst wurde ein `postgres.yml` File erstellt um die Datenbank zu starten, dabei wird auch bei jedem starten der Datenbank ein Script ausgeführt welches die Datenbank mit initialen Daten bestückt.

Zusätzlich wurden zwei Datenbanken erstellt `prod` und `test`. Dabei wird `prod` verwendet wenn der Service läuft und `test` nur für die Unit Tests.

## Implementierung
Es wurde grundsätzlich das zur Verfügung gestellte Repository verwendet.
Verändert wurden nur die Environment variablen auf welche zugegriffen (Name der Datenbanken für `prod` und `test`) wird und die zusätzlichen Funktionalitäten.

```Dockerfile
version: '3'

services:
  db:
    image: postgres
    restart: always
    environment:
      POSTGRES_PASSWORD: example
    ports:
      - 5432:5432
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    command: postgres -c max_connections=300
```

### Environment Variablen
Die Unit Tests verwenden jetzt eine andere Datenbank:

```
	a.Initialize(
		os.Getenv("APP_DB_USERNAME"),
		os.Getenv("APP_DB_PASSWORD"),
		os.Getenv("APP_DB_NAME_TEST"))
```

### Funktionalitäten
Es wurde eine Filterung hinzugefügt um nach Produkten filtern zu können.
```Go
func (a *App) filterProducts(w http.ResponseWriter, r *http.Request) {
	nameFilter := r.FormValue("nameFilter")
	minPriceFilter, err := strconv.ParseFloat(r.FormValue("minPriceFilter"), 64)
	maxPriceFilter, err := strconv.ParseFloat(r.FormValue("maxPriceFilter"), 64)

	products, err := filterProducts(a.DB, nameFilter, minPriceFilter, maxPriceFilter)
	if err != nil {
		respondWithError(w, http.StatusInternalServerError, err.Error())
		return
	}

	respondWithJSON(w, http.StatusOK, products)
}
```

```Go
func filterProducts(db *sql.DB, nameFilter string, minPriceFilter, maxPriceFilter float64) ([]product, error) {
	query := "SELECT id, name, price FROM products WHERE 1 = 1"

	args := []interface{}{}

	if nameFilter != "" {
		query += " AND name LIKE $1"
		args = append(args, "%"+nameFilter+"%")
	}

	if minPriceFilter != 0 {
		query += " AND price >= $2"
		args = append(args, minPriceFilter)
	}

	if maxPriceFilter != 0 {
		query += " AND price <= $3"
		args = append(args, maxPriceFilter)
	}

	products := []product{}

	rows, err := db.Query(query, args...)

	if err != nil {
		return nil, err
	}

	for rows.Next() {
		var p product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price); err != nil {
			return nil, err
		}
		products = append(products, p)
	}

	return products, nil
}
```

Weiters wurde eine Funktion implementiert welche die teuersten Produkte zurückgibt.
```Go
func (a *App) getProductsWithHighestPrice(w http.ResponseWriter, r *http.Request) {
	products, err := getProductsWithHighestPrice(a.DB)

	if err != nil {
		switch err {
		case sql.ErrNoRows:
			respondWithError(w, http.StatusNotFound, "Product not found")
		default:
			respondWithError(w, http.StatusInternalServerError, err.Error())
		}
		return
	}

	respondWithJSON(w, http.StatusOK, products)
}
```

```Go
func getProductsWithHighestPrice(db *sql.DB) ([]product, error) {
	rows, err := db.Query("SELECT id, name, price FROM products WHERE price = (SELECT MAX(price) FROM products)")
	if err != nil {
		return nil, err
	}

	products := []product{}

	for rows.Next() {
		var p product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price); err != nil {
			return nil, err
		}
		products = append(products, p)
	}

	return products, nil
}
```

Anschlißend wurden noch die neuen Routen hinzugefügt:
```Go
func (a *App) initializeRoutes() {
	a.Router.HandleFunc("/products", a.getProducts).Methods("GET")
	a.Router.HandleFunc("/filterProducts", a.filterProducts).Methods("GET")
	a.Router.HandleFunc("/getProductsWithHighestPrice", a.getProductsWithHighestPrice).Methods("GET")
	a.Router.HandleFunc("/product", a.createProduct).Methods("POST")
	a.Router.HandleFunc("/product/{id:[0-9]+}", a.getProduct).Methods("GET")
	a.Router.HandleFunc("/product/{id:[0-9]+}", a.updateProduct).Methods("PUT")
	a.Router.HandleFunc("/product/{id:[0-9]+}", a.deleteProduct).Methods("DELETE")
}
```
