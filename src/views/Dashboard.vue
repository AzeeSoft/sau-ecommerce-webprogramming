<template>
    <div id="dashboard" class="d-flex" v-resize="onWindowResized">
        <NavToolbar @hamburgerClicked="onHamburgerClicked">
            <template slot="hamburgerHolder">
                <div class="d-flex align-center" icon>
                    <v-icon
                        v-if="(isOnMobile && showNavDrawer) || (!isOnMobile && hamburgerClicked)"
                    >chevron_left</v-icon>
                    <v-icon v-else>menu</v-icon>
                </div>
            </template>

            <template slot="mobile"></template>
        </NavToolbar>

        <transition name="nav-drawer" appear>
            <v-hover>
                <v-navigation-drawer
                    app
                    floating
                    :temporary="isOnMobile"
                    :mini-variant="!isOnMobile && (!hover && !hamburgerClicked)"
                    :clipped="true"
                    v-model="showNavDrawer"
                    id="navDrawer"
                    slot-scope="{ hover }"
                    :mobile-break-point="mobileBreakPoint"
                >
                    <!-- Add this for permanent clipping (even in mobile)
                        class="mt-5"
                        style="top: 16px; margin-bottom: 32px"
                    -->
                    <v-list class="pt-2">
                        <router-link v-if="!curAccount" :to="{name: 'signIn'}" tag="v-list-tile">
                            <v-list-tile-avatar>
                                <img src="/images/icons/user.png">
                            </v-list-tile-avatar>

                            <v-list-tile-content>
                                <v-list-tile-title>Sign In / Sign Up</v-list-tile-title>
                            </v-list-tile-content>
                        </router-link>
                        <router-link v-else :to="{name: 'myAccount'}" tag="v-list-tile" append>
                            <v-list-tile-avatar>
                                <img src="/images/icons/user.png">
                            </v-list-tile-avatar>

                            <v-list-tile-content>
                                <v-list-tile-title>My Account</v-list-tile-title>
                            </v-list-tile-content>
                        </router-link>

                        <div v-if="isAdmin">
                            <div
                                v-for="(navRouterLinkGroup, index) in adminNavRouterLinks"
                                :key="'admin'+index"
                            >
                                <v-divider></v-divider>
                                <router-link
                                    v-for="navRouterLink in navRouterLinkGroup"
                                    :key="navRouterLink.title"
                                    :to="navRouterLink.to"
                                    tag="v-list-tile"
                                    append
                                >
                                    <v-list-tile-action>
                                        <v-btn icon outline color>
                                            <v-icon medium>{{navRouterLink.icon}}</v-icon>
                                        </v-btn>
                                    </v-list-tile-action>

                                    <v-list-tile-content>
                                        <v-list-tile-title>
                                            <span class>{{navRouterLink.title}}</span>
                                        </v-list-tile-title>
                                    </v-list-tile-content>
                                </router-link>
                            </div>
                        </div>

                        <div v-for="(navRouterLinkGroup, index) in navRouterLinks" :key="index">
                            <v-divider></v-divider>
                            <router-link
                                v-for="navRouterLink in navRouterLinkGroup"
                                v-show="!navRouterLink.condition || navRouterLink.condition()"
                                :key="navRouterLink.title"
                                :to="navRouterLink.to"
                                tag="v-list-tile"
                                append
                            >
                                <v-list-tile-action>
                                    <v-btn icon>
                                        <v-icon medium>{{navRouterLink.icon}}</v-icon>
                                    </v-btn>
                                </v-list-tile-action>

                                <v-list-tile-content>
                                    <v-list-tile-title>{{navRouterLink.title}}</v-list-tile-title>
                                </v-list-tile-content>
                            </router-link>
                        </div>

                        <v-divider></v-divider>

                        <v-list-tile @click="logout()">
                            <v-list-tile-action>
                                <v-btn icon>
                                    <v-icon>exit_to_app</v-icon>
                                </v-btn>
                            </v-list-tile-action>
                            <v-list-tile-content>
                                <v-list-tile-title>Logout</v-list-tile-title>
                            </v-list-tile-content>
                        </v-list-tile>
                    </v-list>
                </v-navigation-drawer>
            </v-hover>
        </transition>

        <v-content class="dashboard-content">
            <v-container class="dashboard-container d-flex" fluid>
                <v-layout class="dashboard-layout d-flex" row justify-center>
                    <v-flex>
                        <transition name="router-view-switch-default" appear mode="out-in">
                            <router-view></router-view>
                        </transition>
                    </v-flex>
                </v-layout>
            </v-container>
        </v-content>
        <v-footer app>
            <v-container pa-0>
                <v-layout>
                    <v-flex>Made with ❤️ by Azee</v-flex>
                    ©️ {{ new Date().getFullYear() }}
                </v-layout>
            </v-container>
        </v-footer>
    </div>
</template>

<script lang="ts">
    import Vue from "vue";
    import Component from "vue-class-component";
    import store from "@/store";
    import NavToolbar from "@/components/common/navigation/NavToolbar.vue";
    import SnackBar, { SnackBarTypes } from "@/components/singleton/SnackBar.vue";
    import authModule from '@/store/modules/authModule';
    import dashboardModule from '@/store/modules/dashboardModule';
    import { AccountRole } from '@/tools/types/api';

    interface NavRouterLink {
        to: { name: string } | string;
        icon: string;
        title: string;
        addDivider?: boolean;
        condition?: Function;
    }

    @Component({
        components: {
            NavToolbar
        }
    })
    export default class Dashboard extends Vue {
        private mobileBreakPoint = 800;
        private isOnMobile = false;

        /**
         * Actual clicked state of the hamburger icon. Always true during mobile mode.
         */
        private hamburgerClicked = false;

        /**
         * Represents the v-model for the nav drawer. Always true during non-mobile mode.
         */
        private showNavDrawer = true;

        private adminNavRouterLinks: NavRouterLink[][] = [
            [
                {
                    to: { name: 'adminHome' },
                    icon: "dashboard",
                    title: "Admin Panel"
                },
                {
                    to: { name: 'adminAccounts' },
                    icon: "supervised_user_circle",
                    title: "Accounts"
                },
                {
                    to: { name: 'adminVendors' },
                    icon: "local_shipping",
                    title: "Vendors"
                },
                {
                    to: { name: 'adminProducts' },
                    icon: "fastfood",
                    title: "Products"
                },
            ],
        ];

        private navRouterLinks: NavRouterLink[][] = [
            [
                {
                    to: { name: 'home' },
                    icon: "home",
                    title: "Home"
                },
                {
                    to: { name: 'shop' },
                    icon: "store",
                    title: "Shop"
                },
                {
                    to: { name: 'cart' },
                    icon: "shopping_cart",
                    title: "Cart"
                },
                {
                    to: { name: 'orders' },
                    icon: "card_travel",
                    title: "Orders",
                    condition: () => {
                        return authModule.accountData !== null;
                    },
                },
            ],
            [
                {
                    to: { name: 'about' },
                    icon: "info_outline",
                    title: "About"
                }
            ]
        ];

        private get curAccount() {
            return authModule.accountData;
        };

        private get isAdmin() {
            return this.curAccount && this.curAccount.role === AccountRole.Admin;
        };

        private async logout() {
            SnackBar.show("Logging out...");
            const res = await authModule.logoutSession();
            if (res.data.success) {
                SnackBar.show('Logged out.');
                this.goToHomeScreen();
            }
        }

        private goToLoginScreen() {
            this.$router.push({ name: 'signIn' });
        }

        private goToHomeScreen() {
            this.$router.push({ name: 'home' });
        }

        private async mounted() {
            this.onWindowResized();

            dashboardModule.refreshDashboardData();

            await this.validateLoginStatus();
        }

        private async validateLoginStatus() {
            /* const resData = await authService.validateApiToken();
                if (!resData.success) {
                    SnackBar.show('Session expired. Please login again.', SnackBarTypes.Error);
                    this.goToLoginScreen();
                } */
        }

        private onHamburgerClicked() {
            if (!this.isOnMobile) {
                this.hamburgerClicked = !this.hamburgerClicked;
                this.showNavDrawer = !this.isOnMobile || this.hamburgerClicked;
            } else {
                this.hamburgerClicked = true;
                this.showNavDrawer = this.hamburgerClicked;
            }
        }

        private onWindowResized() {
            const wasOnMobile = this.isOnMobile;
            this.isOnMobile = window.innerWidth < this.mobileBreakPoint;

            if (wasOnMobile && !this.isOnMobile) {
                this.hamburgerClicked = this.showNavDrawer;
            }
        }
    }
</script>

<style lang="scss" scoped>
    @import "@/scss/common/constants.scss";

    #dashboard {
        min-height: 100%;

        .dashboard-content {
            flex: 1;

            .dashboard-container {
                min-height: 100%;

                .dashboard-layout {
                    min-height: 100%;
                }
            }
        }
    }

    #navDrawer {
        .v-list {
            .v-list__tile {
                .v-avatar {
                    margin: 4px;
                }

                .v-list__tile__action {
                    .v-btn {
                        margin: 6px;
                    }
                }
            }

            .router-link-active {
                border-right: $accentColor solid $border-small;
                background-color: rgba(127, 127, 127, 0.15);

                .v-list__tile {
                    .v-list__tile__title {
                        font-weight: 600;
                    }
                }
            }
        }
    }

    /* Transitions */

    // Nav Drawer
    .nav-drawer-enter-active,
    .nav-drawer-leave-active {
        transition: transform 0.5s 0.5s !important;
    }
    .nav-drawer-enter,
    .nav-drawer-leave-to {
        transform: translateX(-100px) !important;
    }
</style>
