<script>
    import { onMount, createEventDispatcher } from 'svelte';
    import Drawflow from 'drawflow';
    import 'drawflow/dist/drawflow.min.css';
    import '$lib/drawflow/drawflow.css';
    import { getAgents } from '$lib/services/agent-service.js';
	import { AgentType } from '$lib/helpers/enums';

    let includeRoutingAgent = true;
    let includePlannerAgent = false;
    let includeTaskAgent = false;
    let includeStaticAgent = false;

    /** @type {any[]} */
    let agents = [];

    /** @type {any[]} */
    let agentNodes = [];

    /** @type {import('$agentTypes').AgentFilter} */
	const filter = {
		pager: { page: 1, size: 100, count: 0 },
        disabled: false,
        types: [AgentType.Planning, AgentType.Task]
	};

    /** @type {import('$agentTypes').AgentModel[]} */
    export let routers;

    /** @type {boolean} */
    export let viewOnlyMode = false;

    /** @type {Drawflow} */
    let editor;
    const dispatch = createEventDispatcher();
    
    onMount(async () => {
        if (viewOnlyMode) {
            const response = await getAgents(filter);
            agents = response?.items || [];
        }

        const container = document.getElementById("drawflow");
        if (container) {
            editor = new Drawflow(container);
            editor.reroute = true;
            editor.reroute_fix_curvature = true;
            editor.start();
            editor.on('nodeCreated', id => {
                let node = editor.getNodeFromId(id);
                node.data.nid = id;
                console.log(`Node created ${id} ${node.data.agent}`);
                agentNodes.push(node.data);
            });
            editor.on('nodeSelected', id => {
                console.log("Node selected " + id);
                // emit event
            });
            renderRoutingFlow();
        }
    });    

    function renderRoutingFlow() {
        agentNodes = [];
        editor.clear();
        let posX = 0;
        const nodeSpaceX = 300, nodeSpaceY = 120;

        let posY = routers.length > 0 && !includeTaskAgent ?
            nodeSpaceY * (routers.length + 1) / 2 :
            nodeSpaceY * (agents.length + 1) / 2;

        // add end-user node
        const userNodeId = editor.addNode('user', 0, 1, posX, posY, 'user', 
        {
            id: "",
            profiles: [],
            agent: 'user'
        }, `<i class="mdi mdi-account font-size-16 text-info me-2"></i><span class="h6">User Request</span>`, false);

        // add router node
        posX += nodeSpaceX;
        let routerPosY = viewOnlyMode ? posY : nodeSpaceY * (agents.length > 0 ? agents.length : 1) / (routers.length > 0 ? routers.length : 1);

        routers.forEach(router => {
            /** @type {string[]} */
            let profiles = [];
            const planner = getPlannerName(router);
            const chatTestLinkHtml = router.is_public ? 
                `<a href= "chat/${router.id}" class="btn btn-primary float-end" target="_blank"><i class="bx bx-chat"></i></a>` :
                '';
            let html = `<span class="h5">${router.name} ${chatTestLinkHtml}</span><span class="text-info">Routing with ${planner}</span>`;
            if (router.profiles.length > 0) {
                profiles = router.profiles;
                html += `<br/><i class="mdi mdi-folder font-size-16 text-info me-2"></i><span>${profiles.join(', ')}</span>`;
            }
            
            const data = {
                id: router.id,
                agent: router.name,
                profiles: profiles || [],
                type: router.type,
                routing_rules: router.routing_rules || []
            };

            if (router.is_host) {
                html =`<img src="images/users/bot.png" height="30">${html}`;
            }

            const nodeId = editor.addNode('router', 1, 1, posX, routerPosY, 'router', data, `${html}`, false);;
            // connect user and router
            editor.addConnection(userNodeId, nodeId, `output_1`, `input_1`);    
            routerPosY += nodeSpaceY + 50;
        });

        const plannerAgents = agents.filter(x => x.type === 'planning');
        const otherAgnets = agents.filter(x => x.type !== 'planning');

        posY = 120;
        posX += nodeSpaceX;
        plannerAgents.forEach(agent => {
            /**@type {any} */
            let nodeId = null;
            let html = `<span class="h6">${agent.name}</span>`;

            if (agent.profiles.length > 0) {
                html += `<br/><i class="mdi mdi-folder font-size-16 text-info me-2"></i>` + agent.profiles.join(', ');
            }

            const data = {
                id: agent.id,
                agent: agent.name
            };

            const routers = agentNodes.filter(x => x.type === AgentType.Routing && x.profiles?.includes('planning') && getPlannerName(x) === agent.name);
            if (!viewOnlyMode || routers.length > 0) {
                nodeId = editor.addNode('agent', 1, 0, posX, posY, 'enabled-node', data, html, false);
            }

            routers.forEach(r => {
                if (nodeId) {
                    editor.addConnection(r.nid, nodeId, `output_1`, `input_1`);
                }
            });

            if (!!nodeId) {
                posY += nodeSpaceY;
            }
        });

        posY = viewOnlyMode ? posY + 30 : 120;
        posX += viewOnlyMode ? 0 : nodeSpaceX;
        otherAgnets.forEach(agent => {   
            /**@type {any} */
            let nodeId = null;
            /**@type {any[]} */   
            let profiles = [];

            const chatTestLinkHtml = agent.is_public ? 
                `<a href= "chat/${agent.id}" class="btn btn-primary float-end" target="_blank"><i class="bx bx-chat"></i></a>` : '';
            let html = `<span class="h6">${agent.name}</span>${chatTestLinkHtml}`;
            if (agent.type == AgentType.Static) {
                const taskLinkHtml = `<a href= "page/agent/${agent.id}/task" class="btn btn-primary float-end" target="_blank"><i class="bx bx-task"></i></a>`;
                html += taskLinkHtml;
            }

            if (agent.profiles.length > 0) {
                profiles = agent.profiles;
                html += `<br/><i class="mdi mdi-folder font-size-16 text-info me-2"></i>` + profiles.join(', ');
            }

            if (agent.is_host) {
                html =`<img src="images/users/bot.png" height="30">${html}`;
            }
            
            const data = {
                id: agent.id,
                agent: agent.name,
                profiles: profiles
            };
            

            if (viewOnlyMode && profiles.length > 0) {
                const filteredProfiles = profiles.filter((/** @type {string} */ profile) => profile !== 'planning');
                const foundNodes = agentNodes.filter(ag => ag.type == AgentType.Routing
                                                        && !!ag.profiles?.some((/** @type {any} */ p) => filteredProfiles.includes(p)));

                if (foundNodes.length > 0) {
                    nodeId = editor.addNode('agent', 1, 0, posX, posY, 'enabled-node', data, html, false);
                    filteredProfiles.forEach((/** @type {string} */ profile) => {
                        foundNodes.forEach(r => {
                            if (r.profiles.find((/** @type {string} */ p) => p == profile)) {
                                editor.addConnection(r.nid, nodeId, `output_1`, `input_1`);
                            }
                        });
                    });
                }
            } else if (!viewOnlyMode) {
                nodeId = editor.addNode('agent', 1, 0, posX, posY, 'enabled-node', data, html, false);

                // connect by profile
                if (profiles.length > 0) {
                    // match profile
                    const filteredProfiles = profiles.filter((/** @type {string} */ profile) => profile !== 'planning');
                    filteredProfiles.forEach((/** @type {string} */ profile) => {
                        agentNodes.filter(ag => ag.type == AgentType.Routing).forEach(r => {
                            if (r.profiles.find((/** @type {string} */ p) => p == profile)) {
                                editor.addConnection(r.nid, nodeId, `output_1`, `input_1`);
                            } else {
                                // editor.removeNodeInput(nid, "input_2");
                                // editor.addConnection(userNodeId, nid, `output_1`, `input_1`);
                            }
                        });
                    });
                } else {
                    // profile is empty
                    /*agentNodes.filter(ag => ag.type == "routing" && ag.profiles.length == 0)
                        .forEach(r => {
                            editor.addConnection(r.nid, nid, `output_1`, `input_1`);    
                        });*/
                    
                    editor.addConnection(userNodeId, nodeId, `output_1`, `input_1`);    
                }
            }

            if (!!nodeId) {
                posY += nodeSpaceY;
            }

            // draw fallback routing
            // fallback
            // const fallback = agent.routing_rules.find((/** @type {any} */ p) => p.type == "fallback");
            // if (fallback) {
            //     editor.addNodeOutput(nid);
            //     let router = agentNodes.find(ag => ag.id == fallback.redirectTo);
            //     if (router) {
            //         editor.addNodeInput(router.nid);
            //         var inputs = editor.getNodeFromId(router.nid).inputs;
            //         let inputId = 0;
            //         for (let prop in inputs) {
            //             inputId++;
            //         }
            //         editor.addConnection(nid, router.nid, `output_1`, `input_${inputId}`);
            //     }
            // }
        });
    }
    
    /** @param {import('$agentTypes').AgentModel} router */
    function getPlannerName(router) {
        const planner = router.routing_rules?.find(p => p.type == "planner");
        return !!planner ? planner.field ?? "NaviePlanner" : null;
    }

    async function handlePlannerAgentSelected() {
        includePlannerAgent = !includePlannerAgent;
        filter.types = getAgentTypes();
        const response = await getAgents(filter);
        agents = response?.items || [];
        renderRoutingFlow();
    }

    async function handleTaskAgentSelected() {
        includeTaskAgent = !includeTaskAgent;
        filter.types = getAgentTypes();
        const response = await getAgents(filter);
        agents = response?.items || [];
        renderRoutingFlow();
    }

    async function handleStaticAgentSelected() {
        includeStaticAgent = !includeStaticAgent;
        filter.types = getAgentTypes();
        const response = await getAgents(filter);
        agents = response?.items || [];
        renderRoutingFlow();
    }

    function getAgentTypes() {
        let types = ['none'];
        if (includePlannerAgent) {
            types.push(AgentType.Planning);
        }
        if (includeTaskAgent) {
            types.push(AgentType.Task);
        }
        if (includeStaticAgent) {
            types.push(AgentType.Static);
        }
        return types;
    }

</script>

<div class="btn-group" role="group">
    <input
        type="checkbox"
        class="btn-check active"
        id="btncheck1"
        autocomplete="off"
        disabled={viewOnlyMode}
    />
    <label
        class={`btn btn-${includeRoutingAgent && !viewOnlyMode ? "" : "outline-"}primary`}
        for="btncheck1"
    >
        Routing Agent
    </label>

    <input
        type="checkbox"
        class="btn-check active"
        id="btncheck2"
        autocomplete="off"
        disabled={viewOnlyMode}
        on:click={() => handlePlannerAgentSelected()}
    />
    <label
        class={`btn btn-${includePlannerAgent ? "" : "outline-"}primary`}
        for="btncheck2"
    >
        Planning Agent
    </label>

    <input
        type="checkbox"
        class="btn-check"
        id="btncheck3"
        autocomplete="off"
        disabled={viewOnlyMode}
        on:click={() => handleTaskAgentSelected()}
    />
    <label 
        class={`btn btn-${includeTaskAgent ? "" : "outline-"}primary`}
        for="btncheck3"
    >
        Task Agent
    </label>

    <input
        type="checkbox"
        class="btn-check"
        id="btncheck4"
        autocomplete="off"
        disabled={viewOnlyMode}
        on:click={() => handleStaticAgentSelected()}
    />
    <label
        class={`btn btn-${includeStaticAgent ? "" : "outline-"}primary`}
        for="btncheck4"
    >
        Static Agent
    </label>
</div>

<div id="drawflow" style="height: 72vh; width: 100%"></div>