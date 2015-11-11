.PHONY: cfssl ca req clean

CFSSL	= @env PATH=$(GOPATH)/bin:$(PATH) cfssl
JSON	= env PATH=$(GOPATH)/bin:$(PATH) cfssljson

all: ca req

install-cfssl:
	go get -u -tags nopkcs11 github.com/cloudflare/cfssl/cmd/cfssl
	go get -u github.com/cloudflare/cfssl/cmd/cfssljson

ca:
	mkdir -p certs
	$(CFSSL) gencert -initca config/ca-csr.json | $(JSON) -bare certs/ca

req:
	$(CFSSL) gencert \
	  -ca certs/ca.pem \
	  -ca-key certs/ca-key.pem \
	  -config config/ca-config.json \
	  -hostname core-01,core-02,core-03 \
	  config/req-csr.json | $(JSON) -bare certs/etcd
	$(CFSSL) gencert \
	  -ca certs/ca.pem \
	  -ca-key certs/ca-key.pem \
	  -config config/ca-config.json \
	  -hostname core-01,core-02,core-03 \
	  config/req-csr.json | $(JSON) -bare certs/apiserver
	$(CFSSL) gencert \
	  -ca certs/ca.pem \
	  -ca-key certs/ca-key.pem \
	  -config config/ca-config.json \
	  -hostname core-01,core-02,core-03 \
	  config/req-csr.json | $(JSON) -bare certs/worker
	$(CFSSL) gencert \
	  -ca certs/ca.pem \
	  -ca-key certs/ca-key.pem \
	  -config config/ca-config.json \
	  -hostname core-01,core-02,core-03 \
	  config/req-csr.json | $(JSON) -bare certs/admin

clean:
	rm -rf certs

