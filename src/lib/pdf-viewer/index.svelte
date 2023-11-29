<script>
	import 'pdfjs-dist/web/pdf_viewer.css';
	import { onMount, tick } from 'svelte';

	// Some PDFs need external cmaps.
	//
	const CMAP_URL = new URL('pdfjs-dist/cmaps/', import.meta.url);
	const CMAP_PACKED = true;

	const ENABLE_XFA = true;
	const SEARCH_FOR = ''; // try "Mozilla";

	const SANDBOX_BUNDLE_SRC = new URL('pdfjs-dist//build/pdf.sandbox.mjs', import.meta.url);

	const WHEEL_ZOOM_DISABLED_TIMEOUT = 1000; // ms
	const DEFAULT_SCALE_VALUE = 'page-width';

	export let url;
	export let minScale = 0.25;
	export let maxScale = 4;
	export let onError = (e) => {};

	let pdfjsLib;
	let pdfjsViewer;
	export let mounted = false;

	export let pdfViewer;
	export let eventBus;
	export let pdfLinkService;
	export let pdfFindController;
	export let pdfScriptingManager;

	export let pdfDocument;
	export let loadingTask;

	let _boundEvents = {};
	let zoomSetting = {
		_wheelUnusedTicks: 0,
		_wheelUnusedFactor: 1,
		_touchUnusedTicks: 0,
		_touchUnusedFactor: 1
	};
	let _touchInfo = null;
	let _isCtrlKeyDown = false;

	let supportsMouseWheelZoomCtrlKey = true;
	let supportsMouseWheelZoomMetaKey = true;
	let supportsPinchToZoom = true;

	let viewerContainerElem;

	onMount(async () => {
		// load pdfjs
		pdfjsLib = await import('pdfjs-dist/build/pdf.mjs');
		pdfjsViewer = await import('pdfjs-dist/web/pdf_viewer.mjs');
		const pdfjsWorker = await import('pdfjs-dist/build/pdf.worker.mjs');

		// setup worker
		try {
			pdfjsLib.GlobalWorkerOptions.workerSrc = pdfjsWorker;
			// pdfjs.GlobalWorkerOptions.workerSrc = new URL('pdfjs-dist/build/pdf.worker', import.meta.url);
		} catch (e) {
			console.log(e);
		}

		initViewer();
		handletListener();

		await tick();
		mounted = true;

		return () => {
			removeListener();
		};
	});

	$: mounted, url, mounted && open(url);

	function initViewer() {
		eventBus = new pdfjsViewer.EventBus();

		// (Optionally) enable hyperlinks within PDF files.
		pdfLinkService = new pdfjsViewer.PDFLinkService({
			eventBus
		});

		// (Optionally) enable find controller.
		pdfFindController = new pdfjsViewer.PDFFindController({
			eventBus,
			linkService: pdfLinkService
		});

		// (Optionally) enable scripting support.
		pdfScriptingManager = new pdfjsViewer.PDFScriptingManager({
			eventBus,
			sandboxBundleSrc: SANDBOX_BUNDLE_SRC
		});

		pdfViewer = new pdfjsViewer.PDFViewer({
			container: viewerContainerElem,
			eventBus,
			linkService: pdfLinkService,
			findController: pdfFindController,
			scriptingManager: pdfScriptingManager,
			textLayerMode: 0,
			annotationEditorMode: -1
		});
		pdfLinkService.setViewer(pdfViewer);
		pdfScriptingManager.setViewer(pdfViewer);

		eventBus.on('pagesinit', function () {
			// We can use pdfViewer now, e.g. let's change default scale.
			pdfViewer.currentScaleValue = DEFAULT_SCALE_VALUE;

			// We can try searching for things.
			if (SEARCH_FOR) {
				eventBus.dispatch('find', { type: '', query: SEARCH_FOR });
			}
		});
	}

	function handletListener() {
		function addWindowResolutionChange(evt = null) {
			if (evt) {
				webViewerResolutionChange(evt);
			}
			const mediaQueryList = window.matchMedia(`(resolution: ${window.devicePixelRatio || 1}dppx)`);
			mediaQueryList.addEventListener('change', addWindowResolutionChange, {
				once: true
			});

			if (typeof PDFJSDev !== 'undefined' && PDFJSDev.test('MOZCENTRAL')) {
				return;
			}
			_boundEvents.removeWindowResolutionChange ||= function () {
				mediaQueryList.removeEventListener('change', addWindowResolutionChange);
				_boundEvents.removeWindowResolutionChange = null;
			};
		}
		addWindowResolutionChange();

		_boundEvents.windowResize = () => {
			eventBus.dispatch('resize', { source: window });
		};
		_boundEvents.windowHashChange = () => {
			eventBus.dispatch('hashchange', {
				source: window,
				hash: document.location.hash.substring(1)
			});
		};

		viewerContainerElem.addEventListener('visibilitychange', webViewerVisibilityChange);
		viewerContainerElem.addEventListener('wheel', webViewerWheel, { passive: false });
		viewerContainerElem.addEventListener('touchstart', webViewerTouchStart, {
			passive: false
		});
		viewerContainerElem.addEventListener('touchmove', webViewerTouchMove, {
			passive: false
		});
		viewerContainerElem.addEventListener('touchend', webViewerTouchEnd, {
			passive: false
		});
		viewerContainerElem.addEventListener('click', webViewerClick);
		viewerContainerElem.addEventListener('keydown', webViewerKeyDown);
		viewerContainerElem.addEventListener('keyup', webViewerKeyUp);
		viewerContainerElem.addEventListener('resize', _boundEvents.windowResize);
	}

	function removeListener() {
		viewerContainerElem.removeListener('visibilitychange', webViewerVisibilityChange);
		viewerContainerElem.removeListener('wheel', webViewerWheel);
		viewerContainerElem.removeListener('touchstart', webViewerTouchStart);
		viewerContainerElem.removeListener('touchmove', webViewerTouchMove);
		viewerContainerElem.removeListener('touchend', webViewerTouchEnd);
		viewerContainerElem.removeListener('click', webViewerClick);
		viewerContainerElem.removeListener('keydown', webViewerKeyDown);
		viewerContainerElem.removeListener('keyup', webViewerKeyUp);
		viewerContainerElem.removeListener('resize', _boundEvents.windowResize);
	}

	function open(url) {
		// Loading document.
		loadingTask = pdfjsLib.getDocument({
			url: url,
			cMapUrl: CMAP_URL,
			cMapPacked: CMAP_PACKED,
			enableXfa: ENABLE_XFA
		});

		loadingTask.promise
			.then((doc) => {
				pdfDocument = doc;
				// Document loaded, specifying document for the viewer and
				// the (optional) linkService.
				pdfViewer.setDocument(pdfDocument);

				pdfLinkService.setDocument(pdfDocument, null);
			})
			.catch((err) => {
				console.log(err);
				onError(err);
			});
	}

	function webViewerVisibilityChange(evt) {
		if (document.visibilityState === 'visible') {
			// Ignore mouse wheel zooming during tab switches (bug 1503412).
			setZoomDisabledTimeout();
		}
	}

	let zoomDisabledTimeout = null;
	function setZoomDisabledTimeout() {
		if (zoomDisabledTimeout) {
			clearTimeout(zoomDisabledTimeout);
		}
		zoomDisabledTimeout = setTimeout(function () {
			zoomDisabledTimeout = null;
		}, WHEEL_ZOOM_DISABLED_TIMEOUT);
	}

	function webViewerWheel(evt) {
		if (pdfViewer.isInPresentationMode) {
			return;
		}

		// Pinch-to-zoom on a trackpad maps to a wheel event with ctrlKey set to true
		// https://developer.mozilla.org/en-US/docs/Web/API/WheelEvent#browser_compatibility
		// Hence if ctrlKey is true but ctrl key hasn't been pressed then we can
		// infer that we have a pinch-to-zoom.
		// But the ctrlKey could have been pressed outside of the browser window,
		// hence we try to do some magic to guess if the scaleFactor is likely coming
		// from a pinch-to-zoom or not.

		// It is important that we query deltaMode before delta{X,Y}, so that
		// Firefox doesn't switch to DOM_DELTA_PIXEL mode for compat with other
		// browsers, see https://bugzilla.mozilla.org/show_bug.cgi?id=1392460.
		const deltaMode = evt.deltaMode;

		// The following formula is a bit strange but it comes from:
		// https://searchfox.org/mozilla-central/rev/d62c4c4d5547064487006a1506287da394b64724/widget/InputData.cpp#618-626
		let scaleFactor = Math.exp(-evt.deltaY / 100);

		const isBuiltInMac =
			typeof navigator?.platform === 'string' && navigator.platform.includes('Mac');
		const isPinchToZoom =
			evt.ctrlKey &&
			!_isCtrlKeyDown &&
			deltaMode === WheelEvent.DOM_DELTA_PIXEL &&
			evt.deltaX === 0 &&
			(Math.abs(scaleFactor - 1) < 0.05 || isBuiltInMac) &&
			evt.deltaZ === 0;

		if (
			isPinchToZoom ||
			(evt.ctrlKey && supportsMouseWheelZoomCtrlKey) ||
			(evt.metaKey && supportsMouseWheelZoomMetaKey)
		) {
			// Only zoom the pages, not the entire viewer.
			evt.preventDefault();
			// NOTE: this check must be placed *after* preventDefault.
			//if (zoomDisabledTimeout || document.visibilityState === 'hidden' || overlayManager.active) {
			if (zoomDisabledTimeout || document.visibilityState === 'hidden') {
				return;
			}

			const previousScale = pdfViewer.currentScale;
			if (isPinchToZoom && supportsPinchToZoom) {
				scaleFactor = _accumulateFactor(previousScale, scaleFactor, '_wheelUnusedFactor');
				if (scaleFactor < 1) {
					zoomOut(null, scaleFactor);
				} else if (scaleFactor > 1) {
					zoomIn(null, scaleFactor);
				} else {
					return;
				}
			} else {
				const delta = normalizeWheelEventDirection(evt);

				let ticks = 0;
				if (deltaMode === WheelEvent.DOM_DELTA_LINE || deltaMode === WheelEvent.DOM_DELTA_PAGE) {
					// For line-based devices, use one tick per event, because different
					// OSs have different defaults for the number lines. But we generally
					// want one "clicky" roll of the wheel (which produces one event) to
					// adjust the zoom by one step.
					if (Math.abs(delta) >= 1) {
						ticks = Math.sign(delta);
					} else {
						// If we're getting fractional lines (I can't think of a scenario
						// this might actually happen), be safe and use the accumulator.
						ticks = _accumulateTicks(delta, '_wheelUnusedTicks');
					}
				} else {
					// pixel-based devices
					const PIXELS_PER_LINE_SCALE = 30;
					ticks = _accumulateTicks(delta / PIXELS_PER_LINE_SCALE, '_wheelUnusedTicks');
				}

				if (ticks < 0) {
					zoomOut(-ticks);
				} else if (ticks > 0) {
					zoomIn(ticks);
				} else {
					return;
				}
			}

			// After scaling the page via zoomIn/zoomOut, the position of the upper-
			// left corner is restored. When the mouse wheel is used, the position
			// under the cursor should be restored instead.
			_centerAtPos(previousScale, evt.clientX, evt.clientY);
		} else {
			setZoomDisabledTimeout();
		}
	}

	function webViewerTouchStart(evt) {
		if (pdfViewer.isInPresentationMode || evt.touches.length < 2) {
			return;
		}
		evt.preventDefault();

		//if (evt.touches.length !== 2 || overlayManager.active) {
		if (evt.touches.length !== 2) {
			_touchInfo = null;
			return;
		}

		let [touch0, touch1] = evt.touches;
		if (touch0.identifier > touch1.identifier) {
			[touch0, touch1] = [touch1, touch0];
		}
		_touchInfo = {
			touch0X: touch0.pageX,
			touch0Y: touch0.pageY,
			touch1X: touch1.pageX,
			touch1Y: touch1.pageY
		};
	}

	function webViewerTouchMove(evt) {
		if (!_touchInfo || evt.touches.length !== 2) {
			return;
		}

		let [touch0, touch1] = evt.touches;
		if (touch0.identifier > touch1.identifier) {
			[touch0, touch1] = [touch1, touch0];
		}
		const { pageX: page0X, pageY: page0Y } = touch0;
		const { pageX: page1X, pageY: page1Y } = touch1;
		const {
			touch0X: pTouch0X,
			touch0Y: pTouch0Y,
			touch1X: pTouch1X,
			touch1Y: pTouch1Y
		} = _touchInfo;

		if (
			Math.abs(pTouch0X - page0X) <= 1 &&
			Math.abs(pTouch0Y - page0Y) <= 1 &&
			Math.abs(pTouch1X - page1X) <= 1 &&
			Math.abs(pTouch1Y - page1Y) <= 1
		) {
			// Touches are really too close and it's hard do some basic
			// geometry in order to guess something.
			return;
		}

		_touchInfo.touch0X = page0X;
		_touchInfo.touch0Y = page0Y;
		_touchInfo.touch1X = page1X;
		_touchInfo.touch1Y = page1Y;

		if (pTouch0X === page0X && pTouch0Y === page0Y) {
			// First touch is fixed, if the vectors are collinear then we've a pinch.
			const v1X = pTouch1X - page0X;
			const v1Y = pTouch1Y - page0Y;
			const v2X = page1X - page0X;
			const v2Y = page1Y - page0Y;
			const det = v1X * v2Y - v1Y * v2X;
			// 0.02 is approximatively sin(0.15deg).
			if (Math.abs(det) > 0.02 * Math.hypot(v1X, v1Y) * Math.hypot(v2X, v2Y)) {
				return;
			}
		} else if (pTouch1X === page1X && pTouch1Y === page1Y) {
			// Second touch is fixed, if the vectors are collinear then we've a pinch.
			const v1X = pTouch0X - page1X;
			const v1Y = pTouch0Y - page1Y;
			const v2X = page0X - page1X;
			const v2Y = page0Y - page1Y;
			const det = v1X * v2Y - v1Y * v2X;
			if (Math.abs(det) > 0.02 * Math.hypot(v1X, v1Y) * Math.hypot(v2X, v2Y)) {
				return;
			}
		} else {
			const diff0X = page0X - pTouch0X;
			const diff1X = page1X - pTouch1X;
			const diff0Y = page0Y - pTouch0Y;
			const diff1Y = page1Y - pTouch1Y;
			const dotProduct = diff0X * diff1X + diff0Y * diff1Y;
			if (dotProduct >= 0) {
				// The two touches go in almost the same direction.
				return;
			}
		}

		evt.preventDefault();

		const distance = Math.hypot(page0X - page1X, page0Y - page1Y) || 1;
		const pDistance = Math.hypot(pTouch0X - pTouch1X, pTouch0Y - pTouch1Y) || 1;
		const previousScale = pdfViewer.currentScale;
		if (supportsPinchToZoom) {
			const newScaleFactor = _accumulateFactor(
				previousScale,
				distance / pDistance,
				'_touchUnusedFactor'
			);
			if (newScaleFactor < 1) {
				zoomOut(null, newScaleFactor);
			} else if (newScaleFactor > 1) {
				zoomIn(null, newScaleFactor);
			} else {
				return;
			}
		} else {
			const PIXELS_PER_LINE_SCALE = 30;
			const ticks = _accumulateTicks(
				(distance - pDistance) / PIXELS_PER_LINE_SCALE,
				'_touchUnusedTicks'
			);
			if (ticks < 0) {
				zoomOut(-ticks);
			} else if (ticks > 0) {
				zoomIn(ticks);
			} else {
				return;
			}
		}

		_centerAtPos(previousScale, (page0X + page1X) / 2, (page0Y + page1Y) / 2);
	}

	function webViewerTouchEnd(evt) {
		if (!_touchInfo) {
			return;
		}

		evt.preventDefault();
		_touchInfo = null;
		zoomSetting._touchUnusedTicks = 0;
		zoomSetting._touchUnusedFactor = 1;
	}

	function webViewerClick(evt) {
		if (!secondaryToolbar?.isOpen) {
			return;
		}
	}

	function webViewerKeyUp(evt) {
		// evt.ctrlKey is false hence we use evt.key.
		if (evt.key === 'Control') {
			_isCtrlKeyDown = false;
		}
	}

	function webViewerKeyDown(evt) {
		_isCtrlKeyDown = evt.key === 'Control';

		if (overlayManager.active) {
			return;
		}
		const isViewerInPresentationMode = pdfViewer.isInPresentationMode;

		let handled = false,
			ensureViewerFocused = false;
		const cmd =
			(evt.ctrlKey ? 1 : 0) | (evt.altKey ? 2 : 0) | (evt.shiftKey ? 4 : 0) | (evt.metaKey ? 8 : 0);

		// First, handle the key bindings that are independent whether an input
		// control is selected or not.
		if (cmd === 1 || cmd === 8 || cmd === 5 || cmd === 12) {
			// either CTRL or META key with optional SHIFT.
			switch (evt.keyCode) {
				case 70: // f
					if (!supportsIntegratedFind && !evt.shiftKey) {
						findBar?.open();
						handled = true;
					}
					break;
				case 71: // g
					if (!supportsIntegratedFind) {
						const { state } = findController;
						if (state) {
							const newState = {
								source: window,
								type: 'again',
								findPrevious: cmd === 5 || cmd === 12
							};
							eventBus.dispatch('find', { ...state, ...newState });
						}
						handled = true;
					}
					break;
				case 61: // FF/Mac '='
				case 107: // FF '+' and '='
				case 187: // Chrome '+'
				case 171: // FF with German keyboard
					zoomIn();
					handled = true;
					break;
				case 173: // FF/Mac '-'
				case 109: // FF '-'
				case 189: // Chrome '-'
					zoomOut();
					handled = true;
					break;
				case 48: // '0'
				case 96: // '0' on Numpad of Swedish keyboard
					if (!isViewerInPresentationMode) {
						// keeping it unhandled (to restore page zoom to 100%)
						setTimeout(function () {
							// ... and resetting the scale after browser adjusts its scale
							zoomReset();
						});
						handled = false;
					}
					break;

				case 38: // up arrow
					if (isViewerInPresentationMode || page > 1) {
						page = 1;
						handled = true;
						ensureViewerFocused = true;
					}
					break;
				case 40: // down arrow
					if (isViewerInPresentationMode || page < pagesCount) {
						page = pagesCount;
						handled = true;
						ensureViewerFocused = true;
					}
					break;
			}
		}

		// if (typeof PDFJSDev === 'undefined' || PDFJSDev.test('GENERIC || CHROME')) {
		// 	// CTRL or META without shift
		// 	if (cmd === 1 || cmd === 8) {
		// 		switch (evt.keyCode) {
		// 			case 83: // s
		// 				eventBus.dispatch('download', { source: window });
		// 				handled = true;
		// 				break;

		// 			case 79: // o
		// 				if (typeof PDFJSDev === 'undefined' || PDFJSDev.test('GENERIC')) {
		// 					eventBus.dispatch('openfile', { source: window });
		// 					handled = true;
		// 				}
		// 				break;
		// 		}
		// 	}
		// }

		// CTRL+ALT or Option+Command
		if (cmd === 3 || cmd === 10) {
			switch (evt.keyCode) {
				case 80: // p
					requestPresentationMode();
					handled = true;
					externalServices.reportTelemetry({
						type: 'buttons',
						data: { id: 'presentationModeKeyboard' }
					});
					break;
				case 71: // g
					// focuses input#pageNumber field
					if (appConfig.toolbar) {
						appConfig.toolbar.pageNumber.select();
						handled = true;
					}
					break;
			}
		}

		if (handled) {
			if (ensureViewerFocused && !isViewerInPresentationMode) {
				pdfViewer.focus();
			}
			evt.preventDefault();
			return;
		}

		// Some shortcuts should not get handled if a control/input element
		// is selected.
		const curElement = getActiveOrFocusedElement();
		const curElementTagName = curElement?.tagName.toUpperCase();
		if (
			curElementTagName === 'INPUT' ||
			curElementTagName === 'TEXTAREA' ||
			curElementTagName === 'SELECT' ||
			(curElementTagName === 'BUTTON' &&
				(evt.keyCode === /* Enter = */ 13 || evt.keyCode === /* Space = */ 32)) ||
			curElement?.isContentEditable
		) {
			// Make sure that the secondary toolbar is closed when Escape is pressed.
			if (evt.keyCode !== /* Esc = */ 27) {
				return;
			}
		}

		// No control key pressed at all.
		if (cmd === 0) {
			let turnPage = 0,
				turnOnlyIfPageFit = false;
			switch (evt.keyCode) {
				case 38: // up arrow
				case 33: // pg up
					// vertical scrolling using arrow/pg keys
					if (pdfViewer.isVerticalScrollbarEnabled) {
						turnOnlyIfPageFit = true;
					}
					turnPage = -1;
					break;
				case 8: // backspace
					if (!isViewerInPresentationMode) {
						turnOnlyIfPageFit = true;
					}
					turnPage = -1;
					break;
				case 37: // left arrow
					// horizontal scrolling using arrow keys
					if (pdfViewer.isHorizontalScrollbarEnabled) {
						turnOnlyIfPageFit = true;
					}
				/* falls through */
				case 75: // 'k'
				case 80: // 'p'
					turnPage = -1;
					break;
				case 27: // esc key
					if (secondaryToolbar?.isOpen) {
						secondaryToolbar.close();
						handled = true;
					}
					if (!supportsIntegratedFind && findBar?.opened) {
						findBar.close();
						handled = true;
					}
					break;
				case 40: // down arrow
				case 34: // pg down
					// vertical scrolling using arrow/pg keys
					if (pdfViewer.isVerticalScrollbarEnabled) {
						turnOnlyIfPageFit = true;
					}
					turnPage = 1;
					break;
				case 13: // enter key
				case 32: // spacebar
					if (!isViewerInPresentationMode) {
						turnOnlyIfPageFit = true;
					}
					turnPage = 1;
					break;
				case 39: // right arrow
					// horizontal scrolling using arrow keys
					if (pdfViewer.isHorizontalScrollbarEnabled) {
						turnOnlyIfPageFit = true;
					}
				/* falls through */
				case 74: // 'j'
				case 78: // 'n'
					turnPage = 1;
					break;

				case 36: // home
					if (isViewerInPresentationMode || page > 1) {
						page = 1;
						handled = true;
						ensureViewerFocused = true;
					}
					break;
				case 35: // end
					if (isViewerInPresentationMode || page < pagesCount) {
						page = pagesCount;
						handled = true;
						ensureViewerFocused = true;
					}
					break;

				case 83: // 's'
					// pdfCursorTools?.switchTool(CursorTool.SELECT);
					break;
				case 72: // 'h'
					// pdfCursorTools?.switchTool(CursorTool.HAND);
					break;

				case 82: // 'r'
					// rotatePages(90);
					break;

				case 115: // F4
					pdfSidebar?.toggle();
					break;
			}

			if (turnPage !== 0 && (!turnOnlyIfPageFit || pdfViewer.currentScaleValue === 'page-fit')) {
				if (turnPage > 0) {
					pdfViewer.nextPage();
				} else {
					pdfViewer.previousPage();
				}
				handled = true;
			}
		}

		// shift-key
		if (cmd === 4) {
			switch (evt.keyCode) {
				case 13: // enter key
				case 32: // spacebar
					if (!isViewerInPresentationMode && pdfViewer.currentScaleValue !== 'page-fit') {
						break;
					}
					pdfViewer.previousPage();

					handled = true;
					break;

				case 82: // 'r'
					rotatePages(-90);
					break;
			}
		}

		if (!handled && !isViewerInPresentationMode) {
			// 33=Page Up  34=Page Down  35=End    36=Home
			// 37=Left     38=Up         39=Right  40=Down
			// 32=Spacebar
			if (
				(evt.keyCode >= 33 && evt.keyCode <= 40) ||
				(evt.keyCode === 32 && curElementTagName !== 'BUTTON')
			) {
				ensureViewerFocused = true;
			}
		}

		if (ensureViewerFocused && !pdfViewer.containsElement(curElement)) {
			// The page container is not focused, but a page navigation key has been
			// pressed. Change the focus to the viewer container to make sure that
			// navigation by keyboard works as expected.
			pdfViewer.focus();
		}

		if (handled) {
			evt.preventDefault();
		}
	}

	function _accumulateTicks(ticks, prop) {
		// If the direction changed, reset the accumulated ticks.
		if ((zoomSetting[prop] > 0 && ticks < 0) || (zoomSetting[prop] < 0 && ticks > 0)) {
			this[prop] = 0;
		}
		zoomSetting[prop] += ticks;
		const wholeTicks = Math.trunc(zoomSetting[prop]);
		zoomSetting[prop] -= wholeTicks;
		return wholeTicks;
	}

	function _accumulateFactor(previousScale, factor, prop) {
		if (factor === 1) {
			return 1;
		}
		// If the direction changed, reset the accumulated factor.
		if ((zoomSetting[prop] > 1 && factor < 1) || (zoomSetting[prop] < 1 && factor > 1)) {
			zoomSetting[prop] = 1;
		}

		const newFactor =
			Math.floor(previousScale * factor * zoomSetting[prop] * 100) / (100 * previousScale);
		zoomSetting[prop] = factor / newFactor;

		return newFactor;
	}

	function _centerAtPos(previousScale, x, y) {
		const scaleDiff = pdfViewer.currentScale / previousScale - 1;
		if (scaleDiff !== 0) {
			const [top, left] = pdfViewer.containerTopLeft;
			pdfViewer.container.scrollLeft += (x - left) * scaleDiff;
			pdfViewer.container.scrollTop += (y - top) * scaleDiff;
		}
	}

	function zoomIn(steps, scaleFactor) {
		if (pdfViewer.isInPresentationMode) {
			return;
		}

		try {
			if (pdfViewer.currentScale >= maxScale) {
				pdfViewer.currentScale = maxScale;
			} else {
				pdfViewer.increaseScale({
					drawingDelay: 400,
					steps,
					scaleFactor
				});
				if (pdfViewer.currentScale > maxScale) {
					pdfViewer.currentScale = maxScale;
				}
			}
		} catch (e) {}
	}

	function zoomOut(steps, scaleFactor) {
		if (pdfViewer.isInPresentationMode) {
			return;
		}

		try {
			if (pdfViewer.currentScale <= minScale) {
				pdfViewer.currentScale = minScale;
			} else {
				pdfViewer.decreaseScale({
					drawingDelay: 400,
					steps,
					scaleFactor
				});
				if (pdfViewer.currentScale < minScale) {
					pdfViewer.currentScale = minScale;
				}
			}
		} catch (e) {}
	}

	function zoomReset() {
		if (pdfViewer.isInPresentationMode) {
			return;
		}
		pdfViewer.currentScaleValue = DEFAULT_SCALE_VALUE;
	}

	function normalizeWheelEventDirection(evt) {
		let delta = Math.hypot(evt.deltaX, evt.deltaY);
		const angle = Math.atan2(evt.deltaY, evt.deltaX);
		if (-0.25 * Math.PI < angle && angle < 0.75 * Math.PI) {
			// All that is left-up oriented has to change the sign.
			delta = -delta;
		}
		return delta;
	}

	function requestPresentationMode() {
		// pdfPresentationMode?.request();
	}

	function getActiveOrFocusedElement() {
		let curRoot = document;
		let curActiveOrFocused = curRoot.activeElement || curRoot.querySelector(':focus');

		while (curActiveOrFocused?.shadowRoot) {
			curRoot = curActiveOrFocused.shadowRoot;
			curActiveOrFocused = curRoot.activeElement || curRoot.querySelector(':focus');
		}

		return curActiveOrFocused;
	}
</script>

<div bind:this={viewerContainerElem} id="viewerContainer">
	<div id="viewer" class="pdfViewer"></div>
</div>

<style>
	#viewerContainer {
		overflow: auto;
		position: absolute;
		width: 100%;
		height: 100%;
	}
</style>
