<script>
    import { onMount, onDestroy, afterUpdate } from 'svelte';
    import { derived } from 'svelte/store';
    import { page } from '$app/stores';
    import { _ } from 'svelte-i18n';
    import { Card, CardBody, Col, Row } from '@sveltestrap/sveltestrap';
	import HeadTitle from "$lib/common/HeadTitle.svelte";
	import Breadcrumb from '$lib/common/Breadcrumb.svelte';
	import { globalMenuStore } from '$lib/helpers/store';
	

    /** @type {any} */
    let menuUnsubscribe;

    /** @type {string?} */
    let label = '';

    /** @type {string} */
    let curSlug = '';

    const slug = derived(page, $page => $page.params.reportType);

    const contentSubscribe = slug.subscribe(value => {
        if (curSlug && curSlug !== value) {
            location.reload();
        }
        curSlug = value;
    });

    onMount(async () => {
        menuUnsubscribe = globalMenuStore.subscribe((/** @type {import('$pluginTypes').PluginMenuDefModel[]} */ menu) => {
            const url = getPathUrl();
            let found = menu.find(x => x.link === url);
            label = found?.label || null;
            if (!found?.embeddingInfo) {
                const subFound = menu.find(x => !!x.subMenu?.find(y => y.link === url));
                found = subFound?.subMenu?.find(x => x.link === url);
                label = found?.label || null;
            }
            embed(found?.embeddingInfo || null);
        });
    });

    onDestroy(() => {
        menuUnsubscribe?.();
        contentSubscribe?.();
    });

    
    /** @param {import('$pluginTypes').EmbeddingInfoModel?} data */
    function embed(data) {
        if (!data) return;

        if (data.scriptSrc) {
            const script = document.createElement("script");
            script.id = data.source;
            script.src = data.scriptSrc;
            script.async = true;

            if (data.scriptType) {
                script.type = data.scriptType;
            }
            document.head.appendChild(script);
        }

        const div = document.querySelector('#agent-reporting-content');
        if (!data.url || !div) return;

        if (data.htmlTag) {
            const elem = document.createElement(data.htmlTag);
            elem.id = data.source;
            elem.style.width = '100%';
            elem.style.height = '100%';
            elem.style.justifyContent = 'center';
            // @ts-ignore
            elem.src = data.url;
            div.appendChild(elem);
        }
    }

    const getPathUrl = () => {
		const path = $page.url.pathname;
		return path?.startsWith('/') ? path.substring(1) : path;
	};
</script>

<HeadTitle title="{$_(label || 'Reporting')}" />
<Breadcrumb title="{$_('Agent')}" pagetitle="{$_(label || 'Reporting')}" />

<Row>
	<Col lg="12">
		<Card>
			<CardBody id="agent-reporting-content"></CardBody>
        </Card>
    </Col>
</Row>