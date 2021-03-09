# Unofficial Raindrop.io API client
## Work in progress

[![Actions Status](https://github.com/kattaris/raindrop-io-api-client/workflows/CI/badge.svg)](https://github.com/kattaris/raindrop-io-api-client/actions)
[![Coverage Status](https://codecov.io/github/kattaris/raindrop-io-api-client/coverage.svg?branch=master)](https://codecov.io/gh/kattaris/raindrop-io-api-client)
[![Releases](https://img.shields.io/github/v/release/kattaris/raindrop-io-api-client.svg?include_prereleases&style=flat-square)](https://github.com/kattaris/raindrop-io-api-client/releases)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

### Example usage:

```bigquery
package main

import (
	"fmt"
	"github.com/kattaris/errhand"
	"github.com/kattaris/raindrop-io-api-client/pkg/raindrop"
	"net/http"
	"net/url"
	"os"
	"os/exec"
	"runtime"
	"time"
)

var log *errhand.Errhand

func init() {
	log = errhand.New()
	logPath := os.Getenv("LOGS") + "/raindrop/main.log"
	log.CustomLogger(logPath, "debug")
}

func main() {
	client, err := raindrop.NewClient("client_id",
		"client_secret",
		"http://localhost:8080/oauth")
	log.HandleError(err, true)

	go func() {
		http.HandleFunc("/oauth", client.GetUserCodeHandler)
		err = http.ListenAndServe(":8080", nil)
		log.HandleError(err, true)
	}()

	// Step 1: The authorization request
	authUrl, err := client.GetAuthorizationURL()
	log.HandleError(err, true)

	// Step 2: The redirection to your application site
	u, err := url.QueryUnescape(authUrl.String())
	log.HandleError(err, true)
	err = openBrowser(u)
	log.HandleError(err, true)

	// Step 3: The token exchange
	for client.ClientCode == "" {
		log.Infoln("Waiting for client to authorize")
		time.Sleep(3 * time.Second)
	}

	accessTokenResp, err := client.GetAccessToken(client.ClientCode)
	log.HandleError(err, true)
	accessToken := accessTokenResp.AccessToken

    newColl, err := client.CreateCollection(accessToken, "list",
		"Test", 1, false, 0, nil)
	log.HandleError(err, false)

	collections, err := client.GetCollections(accessToken)
	log.HandleError(err, false)
	
        fmt.Printf("Collections: %v", collections)
	collection := collections.Items[0]
	log.Infoln(collection.Title)
}

func openBrowser(url string) error {
	var err error
	switch runtime.GOOS {
	case "linux":
		err = exec.Command("xdg-open", url).Start()
	case "windows":
		err = exec.Command("rundll32", "url.dll,FileProtocolHandler", url).Start()
	case "darwin":
		err = exec.Command("open", url).Start()
	default:
		err = fmt.Errorf("unsupported platform")
	}

        if err != nil {
		log.Errorln(err)
	}

        return err
}

```