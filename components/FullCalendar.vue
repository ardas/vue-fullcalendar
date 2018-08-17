<template>
    <div ref="calendar" id="calendar"></div>
</template>

<script>
    import defaultsDeep from 'lodash.defaultsdeep'
    import hash from 'object-hash'
    import 'fullcalendar'
    import $ from 'jquery'

    export default {
        props: {
            events: {
                default() {
                    return []
                },
            },

            eventSources: {
                default() {
                    return []
                },
            },

            editable: {
                default() {
                    return true
                },
            },

            selectable: {
                default() {
                    return true
                },
            },

            selectHelper: {
                default() {
                    return true
                },
            },

            header: {
                default() {
                    return {
                        left:   'prev,next today',
                        center: 'title',
                        right:  'month,agendaWeek,agendaDay'
                    }
                },
            },

            defaultView: {
                default() {
                    return 'agendaWeek'
                },
            },

            sync: {
                default() {
                    return false
                }
            },

            config: {
                type: Object,
                default() {
                    return {}
                },
            },
            compareEvents: {
              type: Boolean,
              default: false,
            },
        },

        computed: {
            defaultConfig() {
                const self = this
                return {
                    header: this.header,
                    defaultView: this.defaultView,
                    editable: this.editable,
                    selectable: this.selectable,
                    selectHelper: this.selectHelper,
                    aspectRatio: 2,
                    timeFormat: 'HH:mm',
                    events: this.events,
                    eventSources: this.eventSources,

                    eventRender(...args) {
                        if (this.sync) {
                            self.events = cal.fullCalendar('clientEvents')
                        }
                        self.$emit('event-render', ...args)
                    },

                    eventDestroy(event) {
                        if (this.sync) {
                            self.events = cal.fullCalendar('clientEvents')
                        }
                    },

                    eventClick(...args) {
                        self.$emit('event-selected', ...args)
                    },

                    eventDrop(...args) {
                        self.$emit('event-drop', ...args)
                    },

                    eventReceive(...args) {
                        self.$emit('event-receive', ...args)
                    },

                    eventResize(...args) {
                        self.$emit('event-resize', ...args)
                    },

                    dayClick(...args){
                        self.$emit('day-click', ...args)
                    },
                    select(start, end, jsEvent, view, resource) {
                        self.$emit('event-created', {
                            start,
                            end,
                            allDay: !start.hasTime() && !end.hasTime(),
                            view,
                            resource
                        })
                    }
                }
            },
        },

        mounted() {
            const cal = $(this.$el),
                self = this

            this.$on('remove-event', (event) => {
                if(event && event.hasOwnProperty('id')){
                    $(this.$el).fullCalendar('removeEvents', event.id);
                }else{
                    $(this.$el).fullCalendar('removeEvents', event);
                }
            })

            this.$on('rerender-events', () => {
                $(this.$el).fullCalendar('rerenderEvents')
            })

            this.$on('refetch-events', () => {
                $(this.$el).fullCalendar('refetchEvents')
            })

            this.$on('render-event', (event) => {
                $(this.$el).fullCalendar('renderEvent', event)
            })

            this.$on('reload-events', () => {
                $(this.$el).fullCalendar('removeEvents')
                $(this.$el).fullCalendar('addEventSource', this.events)
            })

            this.$on('rebuild-sources', () => {
                $(this.$el).fullCalendar('removeEventSources')
                this.eventSources.map(event => {
                    $(this.$el).fullCalendar('addEventSource', event)
                })
            })

            cal.fullCalendar(defaultsDeep(this.config, this.defaultConfig))
        },

        methods: {
            fireMethod(...options) {
                return $(this.$el).fullCalendar(...options)
            },
            handleEventsRoughly() {
              $(this.$el).fullCalendar('removeEvents')
              $(this.$el).fullCalendar('addEventSource', this.events)
            },
            handleEventsByComparing(updatedEvents) {
              const eventsFromCalendar = $(this.$el).fullCalendar('clientEvents');
              const updatedEventsWithHashes = updatedEvents
                .map(event => ({ ...event, _hash: hash(event) }));

              const removedEvents = eventsFromCalendar
                .filter(existed => !updatedEventsWithHashes
                  .find(created => existed._hash === created._hash)
                );
              
              const hashesToRemove = [...removedEvents].map(e => e._hash);
              const shouldBeRemoved = event => hashesToRemove.includes(event._hash);
              
              const createdEvents = updatedEventsWithHashes
                .filter(event => !eventsFromCalendar
                  .find(existed => existed._hash === event._hash)
                );

              const changedEvents = updatedEventsWithHashes
                .filter((event) => {
                  const isExisted = (
                    !createdEvents.find(created => created._hash === event._hash) &&
                    !removedEvents.find(removed => removed._hash === event._hash)
                  );
              
                  if (!isExisted) {
                    return false;
                  }
              
                  return !eventsFromCalendar
                    .find(existed => existed._hash === event._hash);
                });

              if (hashesToRemove.length) {
                $(this.$el).fullCalendar('removeEvents', shouldBeRemoved);
              }

              if (changedEvents.length) {
                $(this.$el).fullCalendar('updateEvents', changedEvents);
              }

              if (createdEvents.length) {
                $(this.$el).fullCalendar('addEventSource', createdEvents);
              }
            },
        },

        watch: {
            events: {
                deep: true,
                handler(updatedEvents) {
                    if (this.compareEvents) {
                      this.handleEventsByComparing(updatedEvents);
                    } else {
                      this.handleEventsRoughly();
                    }
                },
            },
            eventSources: {
                deep: true,
                handler(val) {
                    this.$emit('rebuild-sources')
                },
            },
        },

        beforeDestroy() {
            this.$off('remove-event')
            this.$off('rerender-events')
            this.$off('refetch-events')
            this.$off('render-event')
            this.$off('reload-events')
            this.$off('rebuild-sources')
        },
    }
</script>
