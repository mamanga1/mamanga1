// -------------------------------------------------------------------------
// MaIA Core - v1.2.0-Sovereign-Embedded
// Copyright (C) 2026, MamangaMaster (Nodo Corrientes, Argentina)
// Desarrollado por y para la Soberanía Tecnológica del Enjambre.
// Todos los derechos reservados bajo Licencia GNU GPLv3.
// "El silicio no se tira, se milita."
// -------------------------------------------------------------------------


## 📦 Arquitectura del Código Fuente & Despliegue

El ecosistema se divide en dos componentes críticos: el **Agente de Campo (Go)** que corre en el hardware del socio, y el **Escudo Inversor (JavaScript)** que corre en los servidores de borde de Cloudflare.

### 1. El Cliente Todo-en-Uno: `main.go`
Creá un archivo llamado `main.go` en tu entorno local y pegá el siguiente código de grado militar:

```go
package main

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"fmt"
	"math/rand"
	"net"
	"net/http"
	"os"
	"os/exec"
	"runtime"
	"strconv"
	"strings"
	"sync"
	"time"
)

const (
	IDRed   = "3713"
	Version = "1.2.0-Sovereign-Embedded"
)

type ConfigNodo struct {
	WorkerURL      string `json:"worker_url"`      
	XeonRPC        string `json:"xeon_rpc"`        
	NebulaIP       string `json:"nebula_ip"`       
	NebulaInterfaz string `json:"nebula_interfaz"` 
	NebulaPuerto   int    `json:"nebula_puerto"`
	UsarSoloNebula bool   `json:"usar_solo_nebula"`
}

type IdentidadSocio struct {
	ID      string `json:"id"`
	Created int64  `json:"created"`
}

type RespuestaAsignacion struct {
	NebulaIP     string `json:"nebula_ip"`     
	Certificado  string `json:"certificado"`   
	LlavePrivada string `json:"llave_privada"`
	CaCert       string `json:"ca_cert"`       
}

type BilleteraSocio struct {
	SocioID        string  `json:"socio_id"`
	TokensMaIA     float64 `json:"tokens_maia"`
	UltimoBloque   int64   `json:"ultimo_bloque_validado"`
	SincronizadaOk bool    `json:"sincronizada_ok"`
}

type ResultadoAuditoria struct {
	URL      string `json:"url"`
	Latencia string `json:"latencia"`
	Status   string `json:"status"`
}

type ReporteCompleto struct {
	Timestamp   int64                `json:"timestamp"`
	SocioID     string               `json:"socio_id"`
	Plataforma  string               `json:"plataforma"`
	Temperatura float64              `json:"temperatura_local"`
	DecisionIA  int                  `json:"decision_ia"`
	Auditorias  []ResultadoAuditoria `json:"auditorias"`
}

type OrdenNodoCentral struct {
	Comando string   `json:"comando"`
	URLs    []string `json:"urls"`
}

type MaiaEnjambre struct {
	Identidad string
	Config    ConfigNodo
	Billetera BilleteraSocio
	NebulaCmd *exec.Cmd
}

func NuevoEnjambre() *MaiaEnjambre {
	enjambre := &MaiaEnjambre{}
	enjambre.cargarConfiguracion()
	enjambre.gestionarID()
	enjambre.inicializarBilletera()
	return enjambre
}

func (m *MaiaEnjambre) cargarConfiguracion() {
	pathConfig := "config_nodo.json"
	m.Config = ConfigNodo{
		WorkerURL:      "[https://escudo.4sk.uk/api/report](https://escudo.4sk.uk/api/report)", 
		XeonRPC:        "[http://10.0.0.100:8545](http://10.0.0.100:8545)", 
		NebulaIP:       "AUTO", 
		NebulaInterfaz: "nebula1",
		NebulaPuerto:   4242,
		UsarSoloNebula: false,
	}

	if file, err := os.ReadFile(pathConfig); err == nil {
		_ = json.Unmarshal(file, &m.Config)
	} else {
		plantilla, _ := json.MarshalIndent(m.Config, "", "  ")
		_ = os.WriteFile(pathConfig, plantilla, 0644)
	}
}

func (m *MaiaEnjambre) gestionarID() {
	path := "identidad_socio.json"
	if _, err := os.Stat(path); os.IsNotExist(err) {
		token := make([]byte, 24)
		rand.Seed(time.Now().UnixNano())
		rand.Read(token)
		hasher := sha256.New()
		hasher.Write(token)
		newID := hex.EncodeToString(hasher.Sum(nil))
		
		idData := IdentidadSocio{ID: newID, Created: time.Now().Unix()}
		file, _ := json.MarshalIndent(idData, "", "  ")
		_ = os.WriteFile(path, file, 0644)
		m.Identidad = newID
		return
	}
	file, _ := os.ReadFile(path)
	var idData IdentidadSocio
	json.Unmarshal(file, &idData)
	m.Identidad = idData.ID
}

func (m *MaiaEnjambre) inicializarBilletera() {
	m.Billetera = BilleteraSocio{SocioID: m.Identidad, TokensMaIA: 0.0, UltimoBloque: 0, SincronizadaOk: false}
}

func (m *MaiaEnjambre) OrquestarYArrancarNebula() {
	if m.Config.NebulaIP != "AUTO" {
		fmt.Printf("[NEBULA] IP estática detectada: %s. Saltando auto-configuración.\n", m.Config.NebulaIP)
		m.lanzarSubprocesoNebula()
		return
	}

	fmt.Println("[ORQUESTADOR] Conectando al Worker para solicitar IP libre y llaves criptográficas...")
	client := &http.Client{Timeout: 10 * time.Second}
	urlAsignador := fmt.Sprintf("%s/api/registrar-nodo?socio=%s", m.Config.WorkerURL, m.Identidad)
	
	resp, err := client.Get(urlAsignador)
	if err != nil {
		fmt.Printf("[WARN ORQUESTADOR] Worker temporalmente inaccesible: %v. Operando sin billetera segura.\n", err)
		return
	}
	defer resp.Body.Close()

	var asignacion RespuestaAsignacion
	if err := json.NewDecoder(resp.Body).Decode(&asignacion); err != nil {
		fmt.Println("[ERR ORQUESTADOR] Error al decodificar credenciales del Worker.")
		return
	}

	m.Config.NebulaIP = asignacion.NebulaIP
	fmt.Printf("[🎁 NEBULA] Canal destrabado. IP asignada: %s\n", m.Config.NebulaIP)

	_ = os.WriteFile("ca.crt", []byte(asignacion.CaCert), 0600)
	_ = os.WriteFile("host.crt", []byte(asignacion.Certificado), 0600)
	_ = os.WriteFile("host.key", []byte(asignacion.LlavePrivada), 0600)

	m.generarYamlNebula()
	m.lanzarSubprocesoNebula()
}

func (m *MaiaEnjambre) generarYamlNebula() {
	yamlContent := fmt.Sprintf(`
pki:
  ca: ca.crt
  cert: host.crt
  key: host.key

static_host_map:
  "10.0.0.100": ["tu-xeon-bunker-public-ip:4242"]

lighthouse:
  am_lighthouse: false
  hosts:
    - "10.0.0.100"

listen:
  host: 0.0.0.0
  port: %d

tun:
  dev: %s

logging:
  level: info
`, m.Config.NebulaPuerto, m.Config.NebulaInterfaz)

	_ = os.WriteFile("nebula_config.yaml", []byte(yamlContent), 0644)
}

func (m *MaiaEnjambre) lanzarSubprocesoNebula() {
	binario := "./nebula"
	if runtime.GOOS == "windows" {
		binario = "nebula.exe"
	}

	if m.NebulaCmd != nil && m.NebulaCmd.Process != nil {
		return 
	}

	m.NebulaCmd = exec.Command(binario, "-config", "nebula_config.yaml")
	logFile, _ := os.OpenFile("nebula_ejecucion.log", os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0666)
	m.NebulaCmd.Stdout = logFile
	m.NebulaCmd.Stderr = logFile

	err := m.NebulaCmd.Start()
	if err != nil {
		fmt.Printf("[🛡️ ALERTA] No se pudo auto-lanzar el demonio Nebula: %v. Si estás en Android, recordá usar la App oficial de Nebula externa.\n", err)
	} else {
		fmt.Println("[OK NEBULA] Subproceso P2P acoplado y encriptando por abajo.")
	}
}

func (m *MaiaEnjambre) SincronizarBilleteraNebula() {
	ifaces, err := net.Interfaces()
	if err != nil {
		return
	}
	operativo := false
	for _, iface := range ifaces {
		if strings.Contains(iface.Name, m.Config.NebulaInterfaz) {
			operativo = true
			break
		}
	}

	if !operativo && runtime.GOOS != "android" {
		fmt.Println("[🔒 BILLETERA] Canal Nebula inactivo. Movimiento de fondos congelado por seguridad.")
		m.Billetera.SincronizadaOk = false
		return
	}

	client := &http.Client{Timeout: 5 * time.Second}
	resp, err := client.Get(m.Config.XeonRPC + "/api/wallet?socio=" + m.Identidad)
	if err != nil {
		fmt.Println("[WARN] Libro contable de la Xeon inaccesible. Sincronización en espera.")
		return
	}
	defer resp.Body.Close()

	var estado BilleteraSocio
	if err := json.NewDecoder(resp.Body).Decode(&estado); err == nil {
		m.Billetera.TokensMaIA = estado.TokensMaIA
		m.Billetera.UltimoBloque = estado.UltimoBloque
		m.Billetera.SincronizadaOk = true
		fmt.Printf("[💰 BILLETERA] Balance: %.4f MaIA | Bloque Consenso: %d\n", m.Billetera.TokensMaIA, m.Billetera.UltimoBloque)
	}
}

func obtenerTemperaturaLocal() (float64, error) {
	archivo, err := os.ReadFile("/sys/class/thermal/thermal_zone0/temp")
	if err == nil {
		tempStr := strings.TrimSpace(string(archivo))
		temp, err := strconv.ParseFloat(tempStr, 64)
		if err == nil {
			return temp / 1000.0, nil
		}
	}
	client := &http.Client{Timeout: 4 * time.Second}
	resp, err := client.Get("[https://wttr.in/Corrientes?format=%t](https://wttr.in/Corrientes?format=%t)")
	if err == nil {
		defer resp.Body.Close()
		body, _ := os.ReadAll(resp.Body)
		clima := strings.NewReplacer("+", "", "°C", "").Replace(string(body))
		if t, err := strconv.ParseFloat(strings.TrimSpace(clima), 64); err == nil {
			return t, nil
		}
	}
	return 25.0, nil
}

func worker(id int, urls <-chan string, resultados chan<- ResultadoAuditoria, wg *sync.WaitGroup) {
	client := &http.Client{Timeout: 5 * time.Second}
	for url := range urls {
		inicio := time.Now()
		resp, err := client.Get(url)
		latencia := time.Since(inicio)

		if err != nil {
			resultados <- ResultadoAuditoria{URL: url, Latencia: "0ms", Status: "TIMEOUT/ERR"}
		} else {
			resp.Body.Close()
			resultados <- ResultadoAuditoria{URL: url, Latencia: latencia.String(), Status: resp.Status}
		}
	}
	wg.Done()
}

func (m *MaiaEnjambre) EjecutarCicloAuditoria(listaUrls []string) []ResultadoAuditoria {
	canalUrls := make(chan string, len(listaUrls))
	canalResultados := make(chan ResultadoAuditoria, len(listaUrls))
	var wg sync.WaitGroup

	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(i, canalUrls, canalResultados, &wg)
	}
	for _, url := range listaUrls {
		canalUrls <- url
	}
	close(canalUrls)
	wg.Wait()
	close(canalResultados)

	var res []ResultadoAuditoria
	for r := range canalResultados {
		res = append(res, r)
	}
	return res
}

func (m *MaiaEnjambre) despacharYEscuchar(reporte ReporteCompleto) {
	paqueteJSON, _ := json.Marshal(reporte)
	client := &http.Client{Timeout: 10 * time.Second}
	
	resp, err := client.Post(m.Config.WorkerURL, "application/json", strings.NewReader(string(paqueteJSON)))
	if err != nil {
		return
	}
	defer resp.Body.Close()

	fmt.Printf("[OK CAPA 2] Reporte público enviado a: %s\n", m.Config.WorkerURL)

	var orden OrdenNodoCentral
	if err := json.NewDecoder(resp.Body).Decode(&orden); err == nil && len(orden.URLs) > 0 {
		fmt.Printf("\n[🚨 EMERGENCIA] Directiva central inyectada. Ejecutando pool de ráfaga...\n")
		resEmergencia := m.RunEmergencia(orden.URLs)
		for _, r := range resEmergencia {
			fmt.Printf(" > Alerta -> Target: %s | Latencia: %s\n", r.URL, r.Latencia)
		}
	}
}

func (m *MaiaEnjambre) RunEmergencia(urls []string) []ResultadoAuditoria {
	canalUrls := make(chan string, len(urls))
	canalResultados := make(chan ResultadoAuditoria, len(urls))
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go worker(i, canalUrls, canalResultados, &wg)
	}
	for _, url := range urls {
		canalUrls <- url
	}
	close(canalUrls)
	wg.Wait()
	close(canalResultados)
	var res []ResultadoAuditoria
	for r := range canalResultados {
		res = append(res, r)
	}
	return res
}

func main() {
	agente := NuevoEnjambre()
	fmt.Printf("[M-CORE] Red MaIA Activa v%s en %s\n", Version, runtime.GOOS)
	fmt.Printf("[INFO] ID Hash de Trinchera: %s\n", agente.Identidad[:8])

	agente.OrquestarYArrancarNebula()

	objetivosRutina := []string{"[https://www.google.com](https://www.google.com)", "[https://www.claro.com.ar](https://www.claro.com.ar)"}

	for {
		temp, _ := obtenerTemperaturaLocal()
		auditorias := agente.EjecutarCicloAuditoria(objetivosRutina)
		
		reporte := ReporteCompleto{
			Timestamp:   time.Now().Unix(),
			SocioID:     agente.Identidad,
			Plataforma:  fmt.Sprintf("%s_%s", runtime.GOOS, runtime.GOARCH),
			Temperatura: temp,
			DecisionIA:  1,
			Auditorias:  auditorias,
		}

		agente.despacharYEscuchar(reporte)
		agente.SincronizarBilleteraNebula()

		rand.Seed(time.Now().UnixNano())
		time.Sleep(30*time.Second + time.Duration(rand.Intn(5))*time.Second)
	}
}
2. El Orquestador de Borde Cloudflare: index.js
Creá este archivo en tu entorno local antes de subirlo con Wrangler o pegarlo directo en la consola gráfica de Cloudflare Workers:

JavaScript
// === MOTOR DE INFRAESTRUCTURA M-CORE WORKER ===

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);

    // Endpoint 1: DHCP de Trinchera (Asignación automática de IP libre de Nebula)
    if (request.method === 'GET' && url.pathname === '/api/registrar-nodo') {
      return await registrarNodoDHCP(url, env);
    }

    // Endpoint 2: Escudo de anonimato para métricas públicas
    if (request.method === 'POST' && url.pathname === '/api/report') {
      return await recibirMétricasAnonimas(request);
    }

    return new Response(JSON.stringify({ error: "Ruta inválida en MaIA Network" }), { 
      status: 404, 
      headers: { 'Content-Type': 'application/json' } 
    });
  }
};

async function registrarNodoDHCP(url, env) {
  const socioId = url.searchParams.get('socio');
  if (!socioId) {
    return new Response(JSON.stringify({ error: 'Falta Socio ID' }), { status: 400 });
  }

  const configuracionGuardada = await env.ips_ocupadas.get(`socio:${socioId}`);
  if (configuracionGuardada) {
    return new Response(configuracionGuardada, {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
  }

  let ultimaIpAsignada = await env.ips_ocupadas.get('config:ultima_ip');
  let proximoNumero = ultimaIpAsignada ? parseInt(ultimaIpAsignada) : 2; 

  if (proximoNumero >= 254) {
    return new Response(JSON.stringify({ error: 'Subred Nebula /24 saturada' }), { status: 500 });
  }

  let nuevoIdNumero = proximoNumero + 1;
  let ipAsignada = `10.0.0.${nuevoIdNumero}`;

  const llavesPrecalculadas = await env.ips_ocupadas.get(`credencial:${ipAsignada}`);
  
  let certificadoHost = "-----BEGIN CERTIFICATE-----\n[POOL_EMPTY_XEON_ERROR]\n-----END CERTIFICATE-----";
  let llavePrivadaHost = "-----BEGIN NEBULA PRIVATE KEY-----\n[POOL_EMPTY]\n-----END NEBULA PRIVATE KEY-----";
  let certificadoRaizCa = "-----BEGIN CERTIFICATE-----\n[NO_CA_LOADED]\n-----END CERTIFICATE-----";

  if (llavesPrecalculadas) {
    const creds = JSON.parse(llavesPrecalculadas);
    certificadoHost = creds.certificado;
    llavePrivadaHost = creds.llave_privada;
    certificadoRaizCa = creds.ca_cert;
  }

  const estructuraAsignacion = {
    nebula_ip: ipAsignada,
    certificado: certificadoHost,
    llave_privada: llavePrivadaHost,
    ca_cert: certificadoRaizCa
  };

  await env.ips_ocupadas.put(`socio:${socioId}`, JSON.stringify(estructuraAsignacion));
  await env.ips_ocupadas.put('config:ultima_ip', nuevoIdNumero.toString());

  return new Response(JSON.stringify(estructuraAsignacion), {
    status: 200,
    headers: { 'Content-Type': 'application/json' }
  });
}

async function recibirMétricasAnonimas(request) {
  try {
    const datos = await request.json();
    
    const ordenRetorno = {
      comando: "ROUTINE_OK",
      urls: [] 
    };

    return new Response(JSON.stringify(ordenRetorno), {
      status: 200,
      headers: { 'Content-Type': 'application/json' }
    });
  } catch (err) {
    return new Response(JSON.stringify({ error: "Malformed Payload" }), { status: 400 });
  }
}

🛠️ Comandos de Compilación Cruzada (Instalación por Sistema)

Si querés generar tus propios binarios optimizados desde el Búnker Xeon, ejecutá las siguientes directivas en la consola según el sistema operativo del nodo de destino:

🐧 Para Nodos Linux (Servidores Dedicados, Netbooks Recuperadas, MiniArch en TV Boxes)
Bash
GOOS=linux GOARCH=amd64 go build -ldflags="-s -w" -o maia_linux_amd64 main.go

Si compilás para micro-hardware ARM (ej: TV Box X96Q o Raspberry Pi):
Bash
GOOS=linux GOARCH=arm64 go build -ldflags="-s -w" -o maia_linux_arm64 main.go

🪟 Para Nodos Windows (Estaciones de Trabajo y Consolas de Socios)
Bash
GOOS=windows GOARCH=amd64 go build -ldflags="-s -w" -o maia_nodo.exe main.go

🤖 Para Nodos Android (Celulares Zombi o Servidores Móviles en Termux)
Podés compilarlo directamente adentro de la app Termux usando su entorno nativo de Go, o inyectarlo cruzado desde la Xeon con:
Bash
GOOS=android GOARCH=arm64 go build -ldflags="-s -w" -o maia_android main.go

💾 Descarga Directa de Binarios Precompilados (Flota Oficial)

Si no querés compilar el código fuente manualmente, podés descargar directamente las compilaciones estáticas optimizadas y firmadas por el Comando Central MaIA desde la sección de Releases de este repositorio o usando wget/curl:

Windows Client (64-bit): [Descargar maia_nodo.exe](https://github.com/mamanga1/MaIA/releases/latest/download/maia_nodo.exe)

Linux Server (Xeon/AMD64): [Descargar maia_linux_amd64](https://github.com/mamanga1/MaIA/releases/latest/download/maia_linux_amd64)

ARM64 Node (MiniArch TV Box / Android Termux): [Descargar maia_android](https://github.com/mamanga1/MaIA/releases/latest/download/maia_android)

💡 Recordatorio de Trinchera: No olvides emparejar el programa con el binario ejecutable oficial de Nebula correspondiente a tu sistema operativo dentro de la misma carpeta de ejecución para habilitar el caño seguro P2P de la billetera.
