import { CheerioCrawler, Dataset } from '@crawlee/cheerio';
import { Actor } from 'apify';

await Actor.init();

const input = await Actor.getInput() ?? {};
const { region, comuna, rol1, rol2, modo_diagnostico } = input;

if (!comuna || !rol1) {
    throw new Error('Faltan parametros: comuna y rol1');
}

const rolPrincipal = String(rol1).replace(/\D/g, '');
const rolSecundario = String(rol2 || '').replace(/\D/g, '') || '0';

console.log(`Consultando SII: region=${region}, comuna=${comuna}, rol=${rolPrincipal}-${rolSecundario}`);

const params = new URLSearchParams({
    cod_region: region || '',
    cod_comuna: comuna,
    rol1: rolPrincipal,
    rol2: rolSecundario,
    op: 'cert'
});
const certUrl = `https://zeus.sii.cl/avalu_cgi/br/brc120.sh?${params.toString()}`;

const proxyConfiguration = await Actor.createProxyConfiguration();

const crawler = new CheerioCrawler({
    proxyConfiguration,
    maxRequestsPerCrawl: 3,
    requestHandlerTimeoutSecs: 60,
    async requestHandler({ $, body, request }) {
        const texto = $('body').text().replace(/\s+/g, ' ').trim();
        if (modo_diagnostico) {
            await Dataset.pushData({
                modo_diagnostico: true,
                url_consultada: request.url,
                html_completo: body.toString(),
                texto_extraido: texto
            });
            return;
        }
        const extraerMonto = (regex) => {
            const m = texto.match(regex);
            if (!m) return null;
            const num = m[1].replace(/\./g, '').replace(/,/g, '').replace(/\D/g, '');
            return num ? parseInt(num, 10) : null;
        };
        const resultado = {
            rol: `${rolPrincipal}-${rolSecundario}`,
            comuna_codigo: comuna,
            encontrado: false,
            avaluo_total: extraerMonto(/Aval[uú]o\s+Total[:\s]*\$?\s*([\d.,]+)/i),
            avaluo_terreno: extraerMonto(/Terreno[:\s]*\$?\s*([\d.,]+)/i),
            avaluo_construcciones: extraerMonto(/Construcci[oó]n(?:es)?[:\s]*\$?\s*([\d.,]+)/i),
            html_preview: texto.slice(0, 500)
        };
        resultado.encontrado = resultado.avaluo_total !== null;
        await Dataset.pushData(resultado);
    }
});

await crawler.run([certUrl]);
await Actor.exit();
