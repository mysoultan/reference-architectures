### Acquire the CA certificates

1. Generate a CA self-signed TLS cert

   > Contoso Bicycle needs to buy CA certificates, their preference is to use two different TLS certificates. The first one is going to be a user-facing EV cert to serve in front of the Azure Application Gateway and another one a standard cert at the Ingress Controller level which will not be user facing.

   > :warning: Do not use the certificates created by these scripts for actual deployments. The self-signed certificates are provided for ease of illustration purposes only. For your cluster, use your organization's requirements for procurement and lifetime management of TLS certificates, even for development purposes.

   Cluster Ingress Controller Wildcard Certificate: `*.aks-ingress.contoso.com`

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out traefik-ingress-internal-aks-ingress-contoso-com-tls.crt -keyout traefik-ingress-internal-aks-ingress-contoso-com-tls.key -subj "/CN=*.aks-ingress.contoso.com/O=Contoso Aks Ingress"
   rootCertWilcardIngressController=$(cat traefik-ingress-internal-aks-ingress-contoso-com-tls.crt | base64 -w 0)
   ```

   Azure Application Gateway Certificate: `bicycle.contoso.com`

   ```bash
   openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out appgw.crt -keyout appgw.key -subj "/CN=bicycle.contoso.com/O=Contoso Bicycle"
   openssl pkcs12 -export -out appgw.pfx -in appgw.crt -inkey appgw.key -passout pass:
   appGatewayListernerCertificate=$(cat appgw.pfx | base64 -w 0)
   ```

-> Navigate: [Azure Active Directory Integration](./03-aad.md)
