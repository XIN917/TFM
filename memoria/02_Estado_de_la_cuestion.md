# Estado de la cuestión

Este capítulo revisa el contexto en el que se inserta el proyecto: cómo se gestionan hoy las comunicaciones de la Administración en las empresas, qué infraestructura ofrece DEHú/LEMA para automatizar su acceso, qué soluciones existen ya —comerciales, académicas o dentro de la propia Administración— para problemas similares, y en qué punto concreto se sitúa la aportación de este trabajo respecto a todas ellas.

**中文：** 本章回顾项目所处的背景：企业目前如何处理行政机关的通信、DEHú/LEMA为自动化访问提供了什么基础设施、针对类似问题已经存在哪些方案（商业的、学术的，或行政系统内部的），以及本项目相对于这些方案，具体贡献落在哪个位置。

---

## Gestión actual de comunicaciones con la Administración

La obligación de relacionarse electrónicamente con la Administración sitúa a empresas como MGS Seguros ante una tarea recurrente y, en el caso de las notificaciones, sujeta a plazos legales. El acceso manual al portal de DEHú descrito en la Introducción no es una peculiaridad de MGS: es el punto de partida habitual de las organizaciones que empiezan a recibir un volumen considerable de envíos electrónicos, y sus limitaciones —carga operativa, riesgo de plazos no atendidos, necesidad de disponer del certificado digital en los puestos de los usuarios— son comunes a cualquier sector.

**中文：** 与行政机关进行电子化沟通的义务，使像MGS Seguros这样的企业面临一项周期性的任务，而且对于通知（notificaciones）而言，还受法律时限约束。Introducción中描述的通过DEHú门户进行的人工访问，并非MGS独有的现象：这是开始收到相当数量电子送达件的组织的普遍起点，其局限性——运营负担、时限漏接的风险、需要在用户工作站上使用数字证书——在各个行业中都是共通的。

La urgencia de resolver estas limitaciones no es solo organizativa, sino legal: una notificación a la que no se accede en diez días naturales desde su puesta a disposición se considera rechazada y el procedimiento continúa (art. 43.2 de la Ley 39/2015 [2]). La ausencia de un mecanismo de vigilancia sistemático no solo retrasa la gestión interna: puede hacer que los plazos legales empiecen a correr sin que la empresa lo sepa, con las consecuencias jurídicas y económicas que ello conlleva.

**中文：** 解决这些局限性的紧迫性不仅是组织层面的，更是法律层面的：一份通知若自送达（puesta a disposición）之日起满十个自然日仍未被访问，即视为拒收，程序照常继续（《Ley 39/2015》第43.2条）。缺乏系统性的监控机制，不仅会拖慢内部处理速度：还可能导致法律时限在企业毫不知情的情况下开始计算，进而带来相应的法律和经济后果。

---

## DEHú/LEMA como infraestructura de administración electrónica

DEHú cuenta con un marco regulatorio: el Real Decreto 203/2021 [3] es la norma que regula el sistema de notificaciones y comunicaciones electrónicas del que forma parte. Tampoco es el primer sistema de notificación electrónica de la Administración española. Desde julio de 2020, DEHú centraliza progresivamente las notificaciones y comunicaciones de los organismos adheridos a través de los puntos de concentración de notificaciones (PUC) de cada Administración [8], y el 26 de junio de 2023 sustituyó definitivamente a su antecesor, el Sistema de Notificaciones Electrónicas – Dirección Electrónica Habilitada (SNE-DEH), operado por la Fábrica Nacional de Moneda y Timbre – Real Casa de la Moneda (FNMT-RCM) al amparo de la Ley 11/2007 [7]. La adhesión de organismos ha sido progresiva: en marzo de 2023 eran más de 8.500 [4] y en mayo del mismo año más de 8.800, de los tres niveles de la Administración (estatal, autonómico y local) [9].

**中文：** DEHú有自己的法规依据：RD 203/2021（《电子化手段下公共部门行为与运作条例》）正是规范DEHú所属的那套电子通知和通信系统的法规。DEHú也不是西班牙行政机关的第一个电子通知系统。自2020年7月起，DEHú通过各行政机关的"通知集中点"（PUC），逐步集中接入各成员机构的通知和通信；并于2023年6月26日正式全面取代其前身——由FNMT-RCM（国家造币印花厂——皇家造币厂）依据2007年法律运营的"电子通知系统——启用电子地址"（SNE-DEH）。（补充说明：FNMT-RCM是西班牙国有机构，除铸币、印制邮票外，还负责签发数字证书，历史上也运营过政府的电子通知系统。）机构的接入是逐步推进的：2023年3月超过8500个，同年5月已超过8800个，覆盖国家、自治区、地方三个层级。

Para los destinatarios que reciben un volumen elevado de envíos y disponen de capacidad técnica para automatizar su consulta, DEHú ofrece la modalidad de **Grandes Destinatarios**, cuyo acceso automatizado se realiza mediante la interfaz de servicios web **LEMA**. El alta requiere un proceso administrativo propio —declaración responsable, entorno de pruebas y posterior paso a producción— independiente del acceso ciudadano ordinario [10]. Desde el punto de vista técnico, LEMA ofrece esencialmente la misma información que el portal, expuesta mediante servicios SOAP con seguridad WS-Security y complementada con metadatos pensados para su tratamiento automatizado por sistemas de terceros [1]. Es, en ese sentido, una infraestructura de *acceso*, no una capa de *interpretación*: resuelve cómo obtener las comunicaciones de forma automática, pero no dice nada sobre cómo clasificarlas o derivarlas.

**中文：** 对于接收送达件数量较大、且具备技术能力实现自动化查询的收件方，DEHú提供**大宗收件人（Grandes Destinatarios）**这一模式，其自动化访问通过**LEMA**这个Web服务接口实现。申请开通需要走一套独立的行政流程——责任声明、测试环境、随后转入生产环境——区别于普通公民的常规访问方式。从技术角度看，LEMA提供的信息与门户网站基本相同，只是通过带WS-Security安全机制的SOAP服务暴露出来，并附加了一些便于第三方系统自动化处理的元数据（依据：集成指南[1]）。从这个意义上说，LEMA是一层"接入"基础设施，不是"解读"层：它解决的是如何自动获取通信，但对如何分类或分派这些通信，完全没有涉及。

---

## Soluciones y enfoques existentes

Con el fin de valorar hasta qué punto el problema abordado en este trabajo está ya resuelto, se ha revisado el panorama en cuatro frentes distintos: herramientas comerciales específicas de DEHú/LEMA, plataformas genéricas de automatización de correspondencia empresarial, literatura sobre procesamiento inteligente de documentos aplicado al sector asegurador, y precedentes de automatización de comunicaciones dentro de la propia Administración.

**中文：** 为了评估本项目所要解决的问题，目前已经被解决到什么程度，这里从四个不同方向审视了现有的方案：专门针对DEHú/LEMA的商业工具、面向企业通用收文自动化的平台、保险行业中智能文档处理相关的文献，以及西班牙行政系统内部已有的通信自动化先例。

### Herramientas comerciales de acceso a DEHú/LEMA

El mercado español cuenta con varios productos especializados en la gestión de notificaciones electrónicas que se integran con DEHú a través de LEMA en nombre del Gran Destinatario: **MsNotifica** [11], **EdasNEO** [12], **GoodOK Notifica** [13] y **ELON ML** [14], entre otros. Su funcionalidad se concentra en la vigilancia continua de los buzones, la descarga y archivo centralizado de las comunicaciones y —en el caso de EdasNEO y GoodOK Notifica— una asignación configurable por cliente o buzón a un gestor responsable. Esta asignación, sin embargo, se define por reglas estáticas (a qué persona corresponde cada cuenta o CIF), no por una lectura del contenido de la comunicación: ninguna de las herramientas revisadas ofrece clasificación semántica del contenido ni integración con un sistema de gestión interno de la empresa destinataria.

**中文：** 西班牙市场上有若干专门做电子通知管理的产品，它们代表大宗收件人通过LEMA与DEHú对接：**MsNotifica**、**EdasNEO**、**GoodOK Notifica**和**ELON ML**等（各自的开发公司见参考文献）。（说明：真正的"大宗收件人"是客户企业本身，这些产品只是代表它去访问。）它们的功能集中在持续监控邮箱、下载并集中归档通信，以及（EdasNEO和GoodOK Notifica的情况）按客户或邮箱配置分派给负责的管理员。不过，这种分派是由静态规则定义的（哪个账户/CIF对应哪个人），而不是通过读取通信内容来判断的：所审查的工具中，没有一个提供对内容的语义分类，也没有一个与收件企业的内部管理系统集成。

### Automatización genérica de correspondencia empresarial

Al margen de DEHú, existe un mercado maduro de plataformas conocidas como *digital mailroom*: soluciones que capturan correspondencia entrante por múltiples canales (correo postal escaneado, correo electrónico, portales, APIs), la clasifican por tipo con una puntuación de confianza, extraen los datos relevantes, derivan a revisión humana los documentos clasificados con baja confianza y enrutan el resultado al sistema o equipo correspondiente [15][16]. Es significativo que uno de estos proveedores incluya explícitamente las *regulatory notifications* entre los tipos de documento que procesa [15], lo que sugiere que el patrón de trabajo (captura → clasificación → revisión → enrutamiento) es aplicable, en principio, al contenido que gestiona este proyecto. Estas plataformas son, no obstante, agnósticas por diseño respecto al origen del documento: están construidas para canales genéricos, no para conectarse a la API específica de un organismo público concreto como DEHú/LEMA, algo coherente con su planteamiento como productos de propósito general.

**中文：** 除DEHú之外，还存在一个成熟的平台市场，业内称为*digital mailroom*：这类方案通过多个渠道（扫描的纸质信件、电子邮件、门户网站、API）捕获来文，按类型分类并给出置信度评分，提取相关数据，把低置信度分类的文档转入人工复核，并将结果路由到相应的系统或团队。值得注意的是，其中一家供应商明确把*regulatory notifications*（监管/行政类通知）列为其处理的文档类型之一，这表明这套工作模式（捕获→分类→复核→路由）原则上可以套用到本项目所处理的内容上。不过，这些平台在设计上对文档来源是不可知的：它们是为通用渠道搭建的，不是为对接某个具体政府机构的API（比如DEHú/LEMA）而设计的——这与其通用型产品的定位是一致的。

### Procesamiento inteligente de documentos en el sector asegurador

El procesamiento inteligente de documentos (*Intelligent Document Processing*, IDP) —la combinación de OCR y modelos de lenguaje para clasificar y extraer información de documentos no estructurados— es la tecnología en la que se apoyan las plataformas de *digital mailroom* descritas en el apartado anterior. Este apartado se centra en su aplicación al sector asegurador, por ser el sector de actividad de MGS Seguros y, por tanto, el más directamente comparable al contexto de este proyecto.

**中文：** 智能文档处理（Intelligent Document Processing，简称**IDP**）——即OCR与语言模型结合、对非结构化文档进行分类和信息提取——正是上一节所述digital mailroom平台所依托的技术。这一节聚焦它在保险行业中的应用，是因为MGS Seguros本身就属于保险行业，选这个行业的案例是为了跟本项目所处的具体场景最直接可比。

El sector ya cuenta con casos de implantación reales: entidades como **Mutua MAZ** —mutua colaboradora con la Seguridad Social nº 11, especializada en accidentes de trabajo y enfermedad profesional— y **MAPFRE** —grupo asegurador multinacional— han incorporado esta tecnología en flujos de reclamaciones y prevención de blanqueo de capitales [17]. La literatura académica reciente respalda esta tendencia con resultados cuantitativos: un estudio sobre flujos de procesamiento de siniestros basados en LLM obtiene una reducción del 42 % en el tiempo de procesamiento frente a sistemas basados en reglas, con derivación automática a revisión humana según la complejidad del caso [18]; otro trabajo aplica LLM a la generación de resúmenes clínicos y justificaciones de siniestros, y la evalúa con métricas estándar de generación de texto [19].

**中文：** 保险行业已经有真实的落地案例：**Mutua MAZ**——西班牙社会保障第11号合作互助保险机构，专门处理工伤事故和职业病——以及**MAPFRE**——跨国保险集团——都已经把这项技术用在理赔流程和反洗钱预防中。（说明：MAPFRE是普通的商业保险集团，Mutua MAZ是与西班牙社会保障体系合作的"互助保险机构"，专门负责工伤类保险。）近期的学术文献也用量化结果支撑了这一趋势：一项关于基于LLM的理赔处理流程的研究显示，相比基于规则的系统，处理时间缩短了42%，并会根据案件复杂度自动分流到人工复核；另一项研究将LLM用于生成临床摘要和理赔说明，并用标准的文本生成评估指标进行了评测。

### Automatización de comunicaciones dentro de la propia Administración Pública española

Un precedente distinto, pero relevante, es que la propia Administración ya automatiza procesos de tratamiento de comunicaciones y notificaciones, aunque en sistemas ajenos a DEHú. El Ministerio de Justicia mantiene, a través de su Centro de Excelencia de Automatización (CEA), un programa de Robotización de Procesos Administrativos (RPA) que incluye, entre otros procesos, el tratamiento automatizado de comunicaciones, notificaciones y recordatorios de diligencias en expedientes de comisiones rogatorias a través de la plataforma **TEMIS**, y el traspaso automático de expedientes entre los sistemas **LexNET** y **REGES** [20]. Los tres sistemas pertenecen a la infraestructura electrónica propia de la Administración de Justicia: se trata, por tanto, de un precedente de automatización dentro de la Administración, no de acceso a DEHú.

**中文：** 一个方向不同、但相关的先例是：行政系统自己已经在自动化处理通信和通知类流程，只不过是在与DEHú无关的系统里。司法部通过其"自动化卓越中心"（CEA），运行着一套行政流程机器人化（RPA）项目，其中包括：通过**TEMIS**平台，对国际司法协助案卷（comisiones rogatorias）中的通信、通知及催办事项进行自动化处理，以及在**LexNET**和**REGES**两个系统之间自动转移案卷。这三个系统都属于司法行政部门自己的电子基础设施：因此这是行政系统内部自动化的先例，而不是访问DEHú的案例。

El programa está implantado en el Tribunal Supremo y la Audiencia Nacional, y ha recibido reconocimiento externo —entre otros, el premio AMETIC 2022 y el premio SS&C Blue Prism 2023 [20]—, lo que indica que no se trata de un piloto aislado, sino de una iniciativa consolidada.

**中文：** 该项目已经落地于最高法院和国家高等法院，并获得过外部认可——包括AMETIC 2022奖，以及SS&C Blue Prism 2023奖——说明这不是一个孤立的试点项目，而是一个成熟的项目。

Este caso, sin embargo, difiere del proyecto en un aspecto estructural: automatiza el traspaso de comunicaciones entre sistemas de la propia Administración, no la recepción e interpretación de esas comunicaciones por parte de una empresa destinataria externa. Es, además, automatización de flujo (RPA clásica, basada en reglas y rutas fijas), sin una capa de clasificación semántica del contenido. La literatura académica española sobre RPA en el sector público es coherente con este perfil: un estudio sobre auditoría de sistemas RPA en la gestión de ayudas y subvenciones públicas [21] analiza los riesgos de este tipo de automatización de flujo, y un trabajo publicado en la *Revista Vasca de Gestión de Personas y Organizaciones Públicas* [22] distingue la **actuación administrativa automatizada (AAA)**, regulada en la Ley 40/2015 —cuyo artículo 41.1 la define como "cualquier acto o actuación realizada íntegramente a través de medios electrónicos por una Administración Pública [...] en la que no haya intervenido de forma directa un empleado público" [23]—, de la **RPA**, que carece de regulación específica pero ya se utiliza en la práctica administrativa.

**中文：** 不过，这个案例在结构上和本项目存在差异：它自动化的是行政系统自己内部的通信转移，不是外部收件企业对这些通信的接收和解读。而且，它是流程性质的自动化（经典RPA，基于固定规则和固定路径），不包含对内容进行语义分类的这一层。西班牙关于公共部门RPA的学术文献也和这个特征一致：一项关于审计公共补贴管理中RPA系统的研究，分析了这类流程自动化的风险；另一篇发表于《巴斯克地区人事与公共组织管理期刊》的论文，区分了由《Ley 40/2015》规范的**自动化行政行为（AAA）**（第41.1条将其定义为"完全通过电子方式实施、且没有公务员直接介入的行政机关的行为或举措"）与**RPA**（没有专门的法律规范，但已在行政实践中被使用）。

### Comparación de enfoques

La tabla siguiente resume, para cada familia de soluciones revisada, si cubre las capacidades centrales del proyecto: captura automatizada desde DEHú/LEMA, clasificación semántica del contenido mediante IA, revisión humana de las clasificaciones de baja confianza y enrutamiento a un sistema de gestión interno.

**中文：** 下表针对每一类方案，总结它是否覆盖本项目的核心能力：从DEHú/LEMA自动捕获、通过AI对内容进行语义分类、对低置信度分类进行人工复核，以及路由到内部管理系统。

| Enfoque | Captura desde DEHú/LEMA | Clasificación semántica (IA) | Revisión humana (baja confianza) | Enrutamiento a sistema interno |
|---|:---:|:---:|:---:|:---:|
| Herramientas DEHú/LEMA | ✓ | ✗ | No aplica | Parcial (asignación por regla estática) |
| *Digital mailroom* genérico | ✗ (canales genéricos) | ✓ | ✓ | ✓ |
| IDP en seguros | ✗ | ✓ | ✓ | ✓ |
| RPA en la Administración española | ✗ (entre sistemas propios) | ✗ | No aplica | ✓ (entre sistemas de la Administración) |
| **Este proyecto** | ✓ | ✓ | ✓ | ✓ |

**中文：** 表格显示：能从DEHú/LEMA自动捕获的方案（现有商业工具）缺少语义分类；具备AI分类与人工复核的方案（digital mailroom、保险业IDP）又不对接DEHú/LEMA。只有本项目同时覆盖四项能力。

---

## Posicionamiento del proyecto

De la revisión anterior se desprende que existen ya soluciones maduras para cada pieza del problema por separado, pero no se ha encontrado, en el material públicamente disponible consultado, ninguna que las integre para este caso concreto: envíos obtenidos de DEHú/LEMA, clasificados mediante IA y derivados a un sistema interno de gestión. Conviene matizar el alcance de esta afirmación: la búsqueda se ha limitado a fuentes públicas (productos comerciales con presencia web, literatura académica indexada, iniciativas administrativas con difusión oficial), y no puede descartarse la existencia de desarrollos internos no publicitados en otras empresas que resuelvan un problema similar, ya que ese tipo de sistemas, por su propia naturaleza, no dejan rastro público. Dentro de MGS Seguros, en cualquier caso, no existe actualmente ningún sistema de este tipo: esa ausencia es, en última instancia, lo que motiva este proyecto.

**中文：** 从上述回顾可以看出，问题的每一个部分分别都已经有成熟的解决方案，但在已查阅的公开资料中，没有找到针对这一具体场景把它们整合起来的方案：从DEHú/LEMA获取送达件、用AI分类、再分派到内部管理系统。这里需要对这个论断的范围做一个说明：本次检索仅限于公开来源（有网络展示的商业产品、被索引的学术文献、有官方公开信息的行政机关项目），不能排除其他企业存在未公开的内部开发、解决了类似问题的可能性，因为这类系统由于其本身的性质，不会留下公开痕迹。无论如何，在MGS Seguros内部，目前不存在任何此类系统：正是这一空白，最终构成了本项目的动机。

El proyecto se sitúa así en la intersección de tres líneas que, por separado, ya están consolidadas: la integración automatizada con DEHú/LEMA (resuelta por las herramientas comerciales de nicho, pero sin capa de interpretación), la clasificación de documentos no estructurados mediante OCR y LLM con revisión humana (resuelta por el ecosistema de *digital mailroom* y por la literatura de IDP en seguros, pero sin conexión a fuentes de la Administración), y la automatización de comunicaciones dentro de los sistemas propios de una organización (con precedente consolidado en la propia Administración, pero limitada a automatización de flujo sin clasificación semántica y en la dirección contraria: entre administraciones, no de la Administración hacia una empresa).

**中文：** 因此，本项目所处的位置，是三条各自已经成熟的路线的交汇点：与DEHú/LEMA的自动化接入（由细分商业工具解决，但没有解读层）、通过OCR和LLM对非结构化文档进行分类并配以人工复核（由digital mailroom生态和保险业IDP文献解决，但没有连接行政机关的数据源）、以及组织自身系统内部通信的自动化（在行政系统内部已有成熟先例，但局限于没有语义分类的流程自动化，而且方向相反：是行政机关之间，而不是行政机关对企业）。

Respecto al grado de autonomía del sistema, conviene fijar su posición dentro del marco legal revisado: el proyecto no constituye una actuación administrativa automatizada en el sentido del artículo 41.1 de la Ley 40/2015 [23] —no es la Administración quien automatiza un acto con efectos jurídicos propios, sino una empresa destinataria quien automatiza su propia gestión interna de las comunicaciones recibidas— y tampoco pretende sustituir por completo la decisión humana: el mecanismo de revisión mantiene la intervención de un empleado en las clasificaciones de baja confianza, en línea con el patrón ya extendido en los sistemas de clasificación con IA revisados en este capítulo.

**中文：** 关于系统的自主程度，有必要在已梳理的法律框架内明确它的定位：本项目不构成《Ley 40/2015》第41.1条意义上的"自动化行政行为"（AAA）——不是行政机关在自动化一项具有自身法律效力的行为，而是一家收件企业在自动化处理自己对已收到通信的内部管理——项目也不打算完全取代人工决策：复核机制会在低置信度分类的情况下保留员工的介入，这与本章审阅过的、已经在AI分类系统中普遍存在的做法是一致的。

Sobre esta base se plantea, además, una exploración que —según la literatura pública consultada— no tiene precedente directo: el uso del protocolo **MCP** (*Model Context Protocol*) para que un agente de inteligencia artificial pueda corregir sus propios errores de clasificación invocando directamente una acción correctora sobre el sistema de Ticketing, sin depender de la disponibilidad de un operador. El elemento novedoso no es la clasificación mediante LLM en sí —ya documentada en los apartados anteriores—, sino la capacidad del agente de ejecutar por sí mismo esa corrección, que es precisamente lo que MCP habilita como protocolo estándar entre agentes y herramientas.

**中文：** 在此基础上，还提出了一项探索——就已查阅的公开文献而言，没有找到直接的先例：使用**MCP协议（Model Context Protocol）**，让AI智能体能够在不依赖操作员在场的情况下，直接对工单系统（Ticketing）调用纠正动作，来修正自己的分类错误。这里的创新点不是"用LLM做分类"本身——前面几节已经证明这有成熟先例——而是智能体自己执行这一纠正的能力，这正是MCP作为智能体与工具之间标准协议所提供的东西。

La literatura reciente sobre autocorrección de agentes ante fallos de herramientas documenta tanto capacidades como limitaciones significativas. Por un lado, se han observado casos reales de autocorrección ante errores de ejecución —por ejemplo, la reformulación automática de una petición tras una llamada mal formada— en despliegues reales de servidores MCP en entornos científicos [24]. Por otro lado, un banco de pruebas (*benchmark*) reciente que evalúa sistemáticamente la recuperación de agentes ante fallos de herramientas encuentra que la tasa de recuperación ante fallos semánticos implícitos —aquellos en los que la herramienta no señala explícitamente el error, sino que devuelve datos corruptos con apariencia válida— es, de media, unos 37 puntos porcentuales inferior a la obtenida ante fallos explícitos. Además, la tolerancia a fallos mejora con el tamaño del modelo unas 3,66 veces más despacio que el rendimiento en la tarea básica, lo que sitúa la replanificación dinámica como un cuello de botella que no se resuelve simplemente aumentando la escala [25]. En ninguno de los trabajos consultados se documenta la corrección de una clasificación semántica errónea mediante este mecanismo: los fallos estudiados son de ejecución de herramientas (parámetros incorrectos, rutas inválidas, resultados inesperados), no de interpretación semántica del contenido de un documento. Por ello, este componente se plantea como una exploración propia del proyecto, no como la adopción de una solución ya validada.

**中文：** 关于智能体针对工具调用失败进行自我修正的近期文献，既记录了能力，也记录了明显的局限性。一方面，在科学计算环境中实际部署的MCP服务器上，已经观察到真实的自我修正案例——比如某次调用参数格式错误后，自动重新构造请求。另一方面，一项系统性评测智能体面对工具失败时恢复能力的基准测试（benchmark）发现：面对"隐性语义型"故障（工具不明确报错，而是返回看似有效、实则已损坏的数据）时，恢复率平均比面对显性故障时低约37个百分点；此外，随着模型规模增大，容错能力的提升速度比基础任务表现慢约3.66倍——这说明"动态重新规划"是一个单纯靠加大模型规模解决不了的瓶颈。在已查阅的文献中，都没有记录用这套机制去纠正"语义分类判断错误"的情况：这些研究关注的都是工具调用层面的失败（参数错误、路径无效、返回结果异常），不是对文档内容的语义解读错误。因此，这一部分被定位为本项目自身的探索性尝试，而不是采用某个已经被验证过的现成方案。

---

**Referencias citadas en este capítulo:** [1]–[4], [7]–[25] — ver [`Referencias.md`](Referencias.md) para el listado completo y actualizado (numeración ya sincronizada con el orden de aparición en la memoria).